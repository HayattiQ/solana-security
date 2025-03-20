# Arbitrary CPI（任意のプログラム呼び出しの悪用）

## 概要

Solanaではプログラムから別のプログラムへの呼び出し（CPI: Cross-Program Invocation）が可能ですが、呼び出し先プログラムIDを検証しないと、本来呼ぶべきでないプログラムを攻撃者に呼ばされる危険があります。

例えば「トークンを転送するためにSPLトークンプログラムを呼ぶ」ときに、プログラムIDのチェックを怠ると、攻撃者がその引数に自前の悪意あるプログラムのIDを差し込むことが可能です。結果、プログラムは知らぬ間に攻撃者の用意した処理を実行し、意図しないロジックでアカウントが操作される恐れがあります。

## 対処法

CPIで呼び出すプログラムIDを必ず確認します。Anchorを使用している場合は、Accounts構造体で例えば`token_program: Program<'info, Token>`のように型を指定すれば、AnchorがそのアカウントのPubkeyがSPLトークンプログラムID（`spl_token::id()`）と一致することを保証します。

手動でCPIを行う場合でも、呼び出し直前に`assert_eq!(cpi_program_id, expected_program_id)`のようなチェックを入れます。また、Anchor提供の`anchor_spl`クレート等のラッパーを使えば、自動的に正しいプログラムを呼ぶので安全です。

要するに、呼び出し先プログラムが想定どおりか確認することで、攻撃者に任意のプログラムを実行させられるリスクを防ぎます。

## 危険なコード例

以下はSPLトークンの`transfer`をCPIで呼び出す処理ですが、token_programのチェックをしていない例です。

```rust
// 不正: token_programのアドレスが本当にSPLトークンか確認していない
pub fn insecure_transfer(ctx: Context<InsecureTransfer>, amount: u64) -> Result<()> {
    // 攻撃者がtoken_programに別のプログラムIDを差し込める可能性がある
    solana_program::program::invoke(
        &spl_token::instruction::transfer(
            ctx.accounts.token_program.key,
            ctx.accounts.source.key,
            ctx.accounts.destination.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?,
        &[
            ctx.accounts.source.to_account_info(),
            ctx.accounts.destination.to_account_info(),
            ctx.accounts.authority.to_account_info(),
        ],
    )
}
```

## 安全なコード例

Anchorを用いる場合、`token_program: Program<'info, Token>`と宣言するだけでOKです。また手動で行う場合は次のようにプログラムIDをチェックします。

```rust
pub fn secure_transfer(ctx: Context<SecureTransfer>, amount: u64) -> Result<()> {
    // 期待するプログラムID（例ではSPLトークンのID）か確認
    if ctx.accounts.token_program.key() != spl_token::ID {
        return Err(ProgramError::IncorrectProgramId);
    }
    // 検証後にCPI呼び出し
    solana_program::program::invoke(
        &spl_token::instruction::transfer(
            ctx.accounts.token_program.key,
            ctx.accounts.source.key,
            ctx.accounts.destination.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?,
        &[ /* 略 */ ],
    )
}
```

Anchorなら、より簡潔に次のように書けます（推奨）:

```rust
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

pub fn transfer_tokens(ctx: Context<TransferTokens>, amount: u64) -> Result<()> {
    // Anchorのtoken::transferを使えば自動で正しいプログラムを呼ぶ
    token::transfer(ctx.accounts.into_transfer_context(), amount)
}

#[derive(Accounts)]
pub struct TransferTokens<'info> {
    pub source: Account<'info, TokenAccount>,
    pub destination: Account<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,  // Program<'info, Token>であるため常に正しいID
}
```

上記ではAnchorがtoken_programのIDをToken（SPLトークン）であることを保証し、`token::transfer`関数内部でも適切にCPI呼び出しを行います。いずれの方法でも、プログラムIDの検証が肝心です。

## 追加情報

Arbitrary CPIの脆弱性は、特に以下のような状況で発生しやすいです：

1. 複数の外部プログラムを呼び出すコード
2. ユーザーが指定したプログラムIDを使用するコード
3. 手動でCPI呼び出しを構築するコード

安全なCPI実装のためのベストプラクティス：

### 1. プログラムIDの厳格な検証

- 常に呼び出し先プログラムIDを既知の定数と比較する
- 動的に決定されるプログラムIDの場合、信頼できるソースから取得する
- 複数のプログラムを呼び出す場合、それぞれのIDを個別に検証する

### 2. Anchorの型安全機能の活用

- `Program<'info, T>`型を使用してプログラムIDを自動検証する
- `anchor_spl`などの公式ラッパーを使用する
- カスタムプログラム型を定義して型安全性を確保する

### 3. CPI呼び出しの制限

- 必要最小限のCPI呼び出しに制限する
- 信頼できるプログラムのみを呼び出す
- 複雑なCPI連鎖を避ける

Arbitrary CPIの脆弱性は、適切なプログラムID検証を実装することで効果的に防止できます。特にAnchorフレームワークを使用する場合、型システムを活用することで多くの検証を自動化できます。


# 📝 監査手順書: Arbitrary CPI（任意のプログラム呼び出しの悪用）

## 📌 監査目的

- プログラムがCPI（Cross-Program Invocation）を通じて外部プログラムを呼び出す際に、呼び出し先プログラムが適切に検証されていることを確認する。
- 攻撃者が任意の悪意あるプログラムを指定できないように保護されていることを保証する。
- Anchorを使わずに、`solana_program::program::invoke` を用いた自前のCPIを実装している場合、その正当な理由を確認し、明示的なコメントを求める。

## 🔎 監査観点

監査者は以下の観点でコードを精査する。

### 1. プログラムIDの明示的な検証

- CPIで呼び出すプログラムIDが期待するプログラムのIDであるかを明示的に確認するコードが存在するか。

### 2. Anchorを利用した型安全なCPI実装の確認

- Anchorを使用している場合は、`Program<'info, T>`型を適切に使い、Anchorの型安全機能がプログラムIDを自動で検証していることを確認する。

### 3. 手動でCPIを行う場合のプログラムID検証の有無と使用理由の確認

- Anchorを使用せず、手動でCPIを行っている場合に、プログラムIDの検証処理がコードに含まれていることを確認する。
- 手動CPIを使用している理由が妥当であるかを確認し、開発者にその理由をコード内コメントで明示することを求める。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Anchor使用時のCPIプログラムID確認

- [ ] CPIに利用されるアカウント定義で`Program<'info, T>`型を適切に指定し、AnchorによるプログラムID検証が行われていることを確認する。

**良い例（Anchor）:**

```rust
#[derive(Accounts)]
pub struct TokenTransfer<'info> {
    #[account(mut)]
    pub source: Account<'info, TokenAccount>,
    #[account(mut)]
    pub destination: Account<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}
```

### ✔️ Step 2: 手動CPI実施時のプログラムID検証と理由確認

- [ ] Anchor未使用または手動でCPIを構築する場合、プログラムIDを明示的にチェックしていることを確認する。
- [ ] 手動の`solana_program::program::invoke`を使用している箇所を特定し、開発者がそれを使う必要性についてコメントで明示していることを確認する。
- [ ] 明確な理由がない場合、監査コメントとしてAnchorを使った実装を推奨する旨を記載する。

**推奨コメント例（手動CPIを使用している箇所）:**

```rust
// 手動invokeを使う理由:
// Anchorではサポートされない特殊な命令を呼ぶため（例：〇〇プログラムの〇〇命令）
```

監査時に理由のコメントがない場合の監査コメント例：
> 「手動でinvokeを使用していますが、AnchorのCPI機能を使わない理由がコード内に記載されていません。明確な理由をコメントで明示してください。」

## 📂 確認するコード箇所（具体的な確認対象）

- CPI呼び出しを行うすべての関数・インストラクション
- AnchorでCPIを行っている場合の`Program<'info, T>`型アカウント定義箇所
- 手動でCPIを呼び出す際の`solana_program::program::invoke`を使用するコード部分
- 手動invokeを使用している箇所の理由コメント有無

## 💬 監査中に確認する質問例

- 「手動のinvokeを使用している理由は何ですか？Anchorを使うことはできませんか？」
- 「手動でCPIを呼ぶ場合、呼び出し先のプログラムIDをどのように検証していますか？」
- 「手動invokeを行っている箇所で、その必要性がコード内コメント等で明確に示されていますか？」

## 🚨 リスク評価基準

監査中に以下を検出した場合は重大な問題として報告する:

- CPIで呼び出すプログラムIDを一切検証していない場合（特に手動呼び出し）
- Anchor使用時に`Program<'info, T>`型を使わず、プログラムIDの自動検証が働いていない場合
- ユーザー入力からプログラムIDを指定可能で、その検証がない場合

軽微だが改善推奨として報告すべき場合：

- Anchorではなく、手動の`solana_program::program::invoke`を使用しているが、その理由がコード内で明示されていない場合（理由の明記を推奨）

## 🛠 推奨する修正方法

- **Anchor使用時:**
  - `Program<'info, T>`型を使用してプログラムIDを自動検証する
  - `anchor_spl`などの公式ラッパーを使用する

- **手動CPI使用時:**
  - プログラムIDを明示的に検証するコードを追加する
  - 手動CPI使用の理由をコメントで明記する
  - 可能な限りAnchorの型安全なCPI機能に移行する

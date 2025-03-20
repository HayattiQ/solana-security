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

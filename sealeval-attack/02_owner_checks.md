# Owner Checks（所有者チェックの欠如）

## 概要

Solanaの各アカウントには「オーナー（所有者プログラム）」が存在します。特にSPLトークンのトークンアカウントなどでは、そのアカウントがどのプログラムによって管理されているか（Owner）が重要です。

オーナーチェックを怠ると、本来SPLトークンプログラムが所有しているべき口座に、攻撃者が同じ形式の偽のアカウント（Ownerが自作プログラムのものなど）を差し替えて渡すことができます。その結果、例えば残高の読み取りや移転処理で不正なデータを参照させられたり、攻撃者の管理下にある口座に対して操作してしまう危険があります。

## 対処法

アカウントの所有プログラムを検証します。具体的には、期待するオーナーと一致するかチェックする必要があります。

Anchorでは、SPLトークンアカウントの場合、`Account<'info, TokenAccount>`型を使えば所有者が自動的にSPLトークンプログラムであることを保証します。また、`#[account(token::mint = ..., token::authority = ...)]`のような構文でそのトークンアカウントのOwnerや関連付けを検証できます。

手動で行う場合でも、プログラム内で`account.owner != expected_program_id`ならエラーとするチェックを入れるべきです。要するに、「このアカウントは本当に○○プログラムが所有しているか？」を常に確認します。

## 危険なコード例

以下のコードでは、token_accountがSPLトークンプログラムに属する正当な口座であるか確認せずに、その残高等を信用して出力しています。攻撃者は任意のAccountInfoをtoken_accountとして渡せるため、例えば偽造したデータを持つ口座を渡しても検知されません：

```rust
// 不正: token_accountのownerやmintをチェックしていない
pub fn insecure_log_balance(ctx: Context<InsecureLogBalance>) -> Result<()> {
    msg!("Token Account {} の残高: {}", 
         ctx.accounts.token_account.key(),
         ctx.accounts.token_account.amount);
    Ok(())
}

#[derive(Accounts)]
pub struct InsecureLogBalance<'info> {
    pub token_account: Account<'info, TokenAccount>,   // Anchorでは型でTokenAccountを要求しているが…
    pub token_account_owner: Signer<'info>,
    pub mint: Account<'info, Mint>,
}
```

上記では`Account<'info, TokenAccount>`型にしているためAnchorが所有者チェックを行ってくれますが、例えば`AccountInfo<'info>`型を使ったり、Ownerを検証しない独自ロジックの場合は危険です。

## 安全なコード例

Anchorを用いる場合、以下のように`token::authority`や`token::mint`の制約を付けることで、token_accountが期待通りのOwner・Mintであることを保証できます：

```rust
#[derive(Accounts)]
pub struct SecureLogBalance<'info> {
    #[account(token::authority = token_account_owner, token::mint = mint)]
    pub token_account: Account<'info, TokenAccount>,  // オーナーとミントを検証
    pub token_account_owner: Signer<'info>,
    pub mint: Account<'info, Mint>,
}
```

上記では、token_accountのOwnerがtoken_account_owner（署名者）であり、なおかつそのトークンアカウントのMintが指定のmintであることをAnchorがチェックします。このように所有者と関連リソース（Mintなど）の両面を確認することで、偽の口座や無関係なプログラムによる口座を排除できます。

## 追加情報

所有者チェックは、特に以下のような状況で重要です：

1. SPLトークンやNFTなど、標準プログラムが管理するアカウントを扱う場合
2. 複数のプログラム間でデータを共有する場合
3. プログラムが他のプログラムを呼び出す（CPI）場合

所有者チェックを行う際の主なポイント：

- トークンアカウントの場合、所有者がSPLトークンプログラム（`spl_token::id()`）であることを確認
- PDAの場合、所有者が期待するプログラム（通常は自分自身のプログラム）であることを確認
- システムアカウントの場合、所有者がシステムプログラム（`system_program::id()`）であることを確認

Anchorを使用する場合、適切な型（`Account<'info, TokenAccount>`など）と属性（`token::authority`、`token::mint`など）を使用することで、これらのチェックを自動化できます。また、`Program<'info, Token>`のような型を使用することで、プログラムIDの検証も自動的に行われます。


# 📝 監査手順書: Owner Checks（所有者チェックの欠如）

---

## 📌 監査目的

- アカウントの所有者（Owner）が期待通りのプログラムであることを保証する。
- 偽造されたアカウントや、不正な所有者プログラムのアカウントによる攻撃を防止するための所有者チェックが適切に行われていることを確認する。

---

## 🔎 監査観点

監査者は以下の観点でコードを精査する。

### 1. SPLトークンアカウントやNFTアカウント等の所有者チェック

- 所有者が `spl_token::id()` 等の適切なプログラムIDであることを確認する。
- Anchor使用時、Account型を正しく使用し、自動的に所有者検証がなされているか確認する。

### 2. PDA（プログラム派生アドレス）の所有者チェック

- PDAが本プログラムを所有者として設定されていることを検証しているか確認する。

### 3. CPI（プログラム間呼び出し）の場合の所有者チェック

- CPI先のアカウント所有者が期待するプログラムIDかを検証していることを確認する。

---

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Accounts構造体の所有者チェック（Anchor使用時）

- [ ] SPLトークンアカウントやMintアカウントはAnchorの `Account<'info, TokenAccount>` や `Account<'info, Mint>` 型を使用し、所有者チェックが暗黙的に行われていることを確認する。
- [ ] 必要に応じて `#[account(token::authority = ..., token::mint = ...)]` などの属性で所有者の関連付けを明示的に検証していることを確認する。

**良い例:**

```rust
#[derive(Accounts)]
pub struct TransferTokens<'info> {
    #[account(mut, token::authority = from_authority, token::mint = mint)]
    pub from: Account<'info, TokenAccount>,
    pub from_authority: Signer<'info>,
    pub mint: Account<'info, Mint>,
}
```

### ✔️ Step 2: Accountsの所有者の明示的検証（Anchor未使用時）

- [ ] `AccountInfo`型を使用する場合、プログラム内でアカウントの `.owner` フィールドを明示的に検証していることを確認する。
- [ ] 検証結果が不一致の場合、適切なエラーを返すことを確認する。

**良い例:**

```rust
if token_account.owner != &spl_token::id() {
    return Err(ErrorCode::InvalidOwner.into());
}
```

### ✔️ Step 3: PDAに関する所有者チェック

- [ ] PDA生成時に、所有者が自プログラム（`ctx.program_id`）になっていることを確認する。

**良い例:**

```rust
let (pda, _) = Pubkey::find_program_address(&[b"vault"], ctx.program_id);
if vault_account.owner != ctx.program_id {
    return Err(ErrorCode::InvalidPDAOwner.into());
}
```

---

## 📂 確認するコード箇所（具体的な確認対象）

- Anchorを使った`Accounts`構造体内での`Account`型の定義箇所
- Anchor未使用時のインストラクション関数内で`AccountInfo`型を扱う箇所
- PDAを生成または検証するすべてのコード箇所
- CPIによって他プログラムを呼び出す際に渡されるアカウントの所有者検証箇所

---

## 💬 監査中に確認する質問例

- 「このトークンアカウントのOwnerがSPLトークンプログラムであることを保証していますか？」
- 「PDAの所有者が本プログラムであることを検証していますか？」
- 「外部プログラム（CPI）への呼び出し時に、アカウントのOwnerを明示的にチェックしていますか？」

---

## 🚨 リスク評価基準

監査中に以下を検出した場合は重大な問題として報告する:

- SPLトークンやMintアカウントの所有者が全く検証されていない
- PDA生成時に所有者が自プログラムか検証されていない
- CPIを呼び出す際に、渡されたアカウントの所有者を一切検証していない
- `AccountInfo`型を使用しているが、所有者のチェックが全く行われていない

---

## 🛠 推奨する修正方法

- **Anchor使用の場合:**
  - 適切な型（`Account<'info, TokenAccount>`など）を使用する
  - `token::authority`、`token::mint`などの属性を追加する

- **Anchor未使用の場合:**
  - 明示的なチェック（`if account.owner != &expected_program_id`）をコードに追加する

- **PDAを使用する場合:**
  - PDAの所有者が自プログラムであることを確認する

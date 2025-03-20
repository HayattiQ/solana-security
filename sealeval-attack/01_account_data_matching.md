# Account Data Matching（アカウントデータの検証不足）

## 概要

アカウントの中身（データ構造上のフィールドなど）が期待通りのものか確認せずに更新や操作を行うと、不正なアカウントを渡された場合に想定外の変更を許してしまいます。

例えば「自分のVaultに対してデータ更新する」機能で、プログラムがVaultアカウントに紐づくオーナーを確認しないと、攻撃者が他人のVaultを自分のものとして引き渡し、そのデータを書き換えることが可能になります。

## 対処法

アカウント内の権限フィールド等を必ず検証し、一致しない場合は処理を中断します。Anchorを使う場合、`#[account(constraint = account.field == expected)]`のようなconstraint属性で検証を宣言できます。

例えばVaultアカウントに格納されたvault_authorityが、関数の引数として渡された権限者のPubkeyと一致するかチェックするコードを入れます。これにより、許可されたユーザ以外が他人のアカウントデータを書き換えるのを防ぎます。

## 危険なコード例

以下の不備な例では、vaultアカウントの所有者確認をせずにデータを書き換えています。このため攻撃者は自分の権限で他人のvaultを引数に渡し、dataを書き換えることができます：

```rust
// 不正: vault_authorityの確認がないため誰のVaultでも更新できてしまう
pub fn update_vault_data_insecure(ctx: Context<UpdateVaultInsecure>, new_data: u8) -> Result<()> {
    let vault = &mut ctx.accounts.vault;
    vault.data = new_data;  // 権限者の確認なくデータを書き換え
    Ok(())
}
```

## 安全なコード例

vaultアカウントに権限者フィールド（例えばvault.authority）がある場合、それが現在の呼び出し元（署名者）のPubkeyと一致することを確認します。またAnchorならhas_oneやconstraintで簡潔に表現できます：

```rust
pub fn update_vault_data_secure(ctx: Context<UpdateVaultSecure>, new_data: u8) -> Result<()> {
    let vault = &mut ctx.accounts.vault;
    if vault.vault_authority != ctx.accounts.vault_authority.key() {
        return Err(ErrorCode::UnauthorizedVaultUpdate.into());  // 権限者が不一致なので中止
    }
    vault.data = new_data;
    Ok(())
}
```

Anchorでは上記チェックと同等のことを、以下のようにアカウント構造体で宣言できます：

```rust
#[derive(Accounts)]
pub struct UpdateVaultSecure<'info> {
    #[account(mut, constraint = vault.vault_authority == vault_authority.key())]
    pub vault: Account<'info, Vault>,
    pub vault_authority: Signer<'info>,
    // ...省略...
}
```

このようにアカウント内部のデータ（権限者や関連フィールド）のマッチング確認を行うことで、不正なアカウントを渡されても安全に拒否できます。

## 追加情報

アカウントデータの検証は、特に以下のような状況で重要です：

1. 複数のユーザーが同じプログラムを使用する場合
2. 異なるアカウント間で関連性がある場合（例：ユーザーとそのデータ）
3. 権限が階層化されている場合（例：管理者と一般ユーザー）

検証すべき主なフィールドには以下のようなものがあります：

- 所有者/権限者のPubkey
- 関連するトークンやNFTのMint
- アカウントの種類や状態を示すフラグ
- タイムスタンプや有効期限

Anchorを使用する場合、`constraint`、`has_one`、`belongs_to`などの属性を活用することで、これらの検証を宣言的に記述でき、コードの可読性と安全性を高めることができます。


# 📝 監査手順書: Account Data Matching（アカウントデータの検証不足）

---

## 📌 監査目的

- アカウント内のデータ（authority、所有者、状態フラグ等）がプログラムの期待するものと一致していることを確認する。
- 不正なアカウントの持ち込みや、他人のアカウントデータの改ざんを防ぐための検証が適切に行われていることを保証する。

---

## 🔎 監査観点

監査者は以下の観点でコードを精査する。

### 1. Accounts構造体でのデータ検証（Anchor使用時）

- Anchorの`constraint`属性、`has_one`属性、`belongs_to`属性などが設定されていることを確認する。
- アカウント内の重要フィールド（authority、所有者Pubkey、状態フラグなど）を期待値と比較していることを確認する。

### 2. インストラクション関数内での明示的なデータ検証（Anchor未使用時）

- アカウント内のauthorityフィールド等が署名者（Signer）や関連アカウントと明示的に比較されていることを確認する。
- 比較結果が不一致の場合、エラー（例: `Unauthorized`）を返していることを確認する。

### 3. 複数アカウント間のデータ整合性検証

- 関連性があるアカウント間で、authority、所有者Pubkey、Mintアドレス等が適切に検証されていることを確認する。
- Anchor使用の場合、`has_one`、`belongs_to`などで宣言的に検証されていることを確認する。

---

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Accounts構造体のconstraint検証（Anchor使用時）

- [ ] すべての`#[derive(Accounts)]`構造体を確認する。
- [ ] データ検証が必要なアカウントに対して`constraint`属性が適切に設定されていることを確認する。
- [ ] 特にauthorityや所有者フィールドが署名者と一致していることを保証する記述があることを確認する。

**良い例:**

```rust
#[account(mut, constraint = vault.vault_authority == vault_authority.key())]
```

### ✔️ Step 2: インストラクション関数内での明示的データ整合性検証（Anchor未使用時）

- [ ] インストラクション関数内でアカウントのauthorityや所有者フィールドを明示的に比較していることを確認する。
- [ ] 比較結果が不一致の場合に適切なエラーが返されていることを確認する。

**危険な例:**

```rust
// vault_authorityの検証なし
pub fn update_vault_data_insecure(ctx: Context<UpdateVaultInsecure>, new_data: u8) -> Result<()> {
    ctx.accounts.vault.data = new_data;
    Ok(())
}
```

**安全な例:**

```rust
// vault_authorityを検証している
pub fn update_vault_data_secure(ctx: Context<UpdateVaultSecure>, new_data: u8) -> Result<()> {
    require!(
        ctx.accounts.vault.vault_authority == ctx.accounts.vault_authority.key(),
        UnauthorizedVaultUpdate
    );
    ctx.accounts.vault.data = new_data;
    Ok(())
}
```

### ✔️ Step 3: 複数アカウント間の整合性チェック

- [ ] アカウント間でauthority、Mintアドレス等の一致が求められる場合、適切な検証が行われていることを確認する。
- [ ] Anchorを使う場合、`has_one`や`belongs_to`などを適切に利用していることを確認する。

**良い例:**

```rust
#[account(mut, has_one = owner)]
```

---

## 📂 確認するコード箇所（具体的な確認対象）

- `#[derive(Accounts)]`構造体内の`constraint`属性の設定状況
- 各インストラクション関数内のauthorityフィールド等の比較箇所
- 重要なアカウントデータの更新処理の直前

---

## 💬 監査中に確認する質問例

- 「このアカウントのauthorityフィールドは署名者と確実に一致していますか？」
- 「アカウントが想定外のデータを持ち込まれた場合、適切に処理が拒否されますか？」
- 「関連アカウント間で重要フィールドが適切に検証されていますか？」

---

## 🚨 リスク評価基準

監査中に以下を検出した場合は重大な問題として報告する:

- アカウントデータの検証が全く存在しない場合
- 権限者の検証が不十分で、悪意ある第三者がデータを書き換え可能な場合
- 関連アカウント間の整合性を一切確認していない場合

---

## 🛠 推奨する修正方法

- **Anchorを使用している場合:**
  - `constraint`、`has_one`、`belongs_to`属性を追加し検証を明確化する。

- **Anchorを使用していない場合:**
  - `require!`マクロや`assert_eq!`を使い、authorityフィールド等を明示的に検証する。

**修正後の推奨コード例（Anchor使用）:**

```rust
#[derive(Accounts)]
pub struct UpdateVaultSecure<'info> {
    #[account(mut, constraint = vault.vault_authority == vault_authority.key())]
    pub vault: Account<'info, Vault>,
    pub vault_authority: Signer<'info>,
}

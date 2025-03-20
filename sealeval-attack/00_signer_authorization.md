# Signer Authorization（署名者チェックの欠如）

## 概要

トランザクションの署名者であること自体に任せきり、プログラム内で適切な署名者の検証を行わないと、想定外のアカウントが権限を持つ操作を実行できてしまいます。つまり「権限を持つはずのアカウント」が実際に署名しているか確認しないと、誰でもその権限を偽装して操作を実行できる恐れがあります。

例えば本来特定ユーザのみ変更できるデータを、悪意ある第三者が署名者チェック抜け穴を利用して改ざんするといった攻撃が可能です。

## 対処法

必ず署名者チェックを実装します。Anchorを使っている場合は、Accounts構造体でSigner<'info>型を用いて署名者であることを要求する他、署名者のPubkeyがアカウント内のauthorityフィールドと一致しているかを検証します。

Anchorではhas_one属性を利用して関連付けを強制することが推奨されます。また、Anchorを使わない場合でもプログラム内で`if account.authority != signer.key()`のようなチェックを行い、不一致ならエラーを返すようにします。

## 危険なコード例

以下の例では、Escrowアカウントの権限者（authority）を検証せずに誰でもデータを書き換えています（署名者であれば通ってしまう）：

```rust
// 不正: escrow.authorityと署名者を照合していない
pub fn insecure_authorization(ctx: Context<InsecureAuthorization>, data: u8) -> Result<()> {
    let escrow = &mut ctx.accounts.escrow;
    escrow.data = data;  // 誰でもデータを書き換え可能
    Ok(())
}

#[derive(Accounts)]
pub struct InsecureAuthorization<'info> {
    pub authority: Signer<'info>,              // 署名者
    #[account(mut, seeds = [b"escrow"], bump)] // has_oneがなく権限検証が不完全
    pub escrow: Account<'info, Escrow>,
}
```

## 安全なコード例

Anchorでは、`#[account(has_one = authority)]`を使用することで、Escrow内のauthorityフィールドが提供された署名者と一致していることを自動チェックできます。また、念のため関数内でも以下のように検証します：

```rust
pub fn secure_authorization(ctx: Context<SecureAuthorization>, data: u8) -> Result<()> {
    let escrow = &mut ctx.accounts.escrow;
    require!(escrow.authority == ctx.accounts.authority.key(), Unauthorized);  // 権限者検証
    escrow.data = data;
    Ok(())
}

#[derive(Accounts)]
pub struct SecureAuthorization<'info> {
    pub authority: Signer<'info>,  // 署名者
    #[account(mut, seeds = [b"escrow"], bump, has_one = authority)]
    pub escrow: Account<'info, Escrow>,  // authorityが署名者と一致することをAnchorが保証
}
```

上記のように署名者の照合チェックを入れることで、想定外のアカウントによる不正実行を防ぎます。

## 追加情報

署名者チェックは、Solanaプログラムのセキュリティにおいて最も基本的かつ重要な要素の一つです。特に以下の点に注意しましょう：

1. 単に署名者であることを確認するだけでなく、その署名者が特定の権限を持っているかを検証する
2. 複数の署名者が関わる場合は、それぞれの役割と権限を明確に区別する
3. PDAsを使用する場合でも、そのPDAに関連付けられた権限者の検証を怠らない

Anchorフレームワークを使用すると、`has_one`や`constraint`などの属性を通じて、これらのチェックを宣言的に記述できるため、人為的なミスを減らすことができます。


# 📝 監査手順書: Signer Authorization（署名者チェックの欠如）

---

## 📌 監査目的

- プログラム内で、操作を実行する署名者が適切に検証されているかを確認する。
- 不適切な署名者がデータや資産に対して変更や操作を行うことができないことを保証する。

---

## 🔎 監査観点

監査者は以下の観点でコードを精査する。

### 1. Accounts構造体の署名者チェック（Anchor使用時）

- `Signer<'info>`型を適切に使っているか。
- Anchorの`has_one`属性や`constraint`属性を使い、署名者とアカウント内の権限者フィールドの整合性を強制的にチェックしているか。

### 2. プログラム内ロジックの署名者チェック（Anchor未使用時）

- 署名者（Signer）のPubkeyがアカウント内の権限者フィールド（authorityなど）と明示的に比較されているか。
- 比較結果が不一致の場合にエラー（例: `Unauthorized`）が返されるか。

### 3. PDA（プログラム派生アドレス）を使用した場合の権限チェック

- PDAのauthorityフィールドが署名者と一致しているか。
- Seedsを用いて適切な署名（signer_seeds）を確認しているか。

---

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Accounts構造体の精査（Anchorの場合）

- [ ] `#[derive(Accounts)]`が付与された構造体をすべて確認する。
- [ ] Accounts構造体内で、権限を要求するアカウント（例: Escrow, Vaultなど）に対し、適切な`has_one`や`constraint`属性が設定されていることを確認する。
- [ ] `Signer<'info>`がAccounts構造体に含まれているか確認する。

**確認例:**

```rust
#[derive(Accounts)]
pub struct SecureAuthorization<'info> {
    pub authority: Signer<'info>,
    #[account(mut, has_one = authority)]  // authorityチェックが明示的に指定されている
    pub escrow: Account<'info, Escrow>,
}

### ✔️ Step 1: Accounts構造体の精査（Anchorの場合）

- [ ] `#[derive(Accounts)]`が付与された構造体をすべて確認する。
- [ ] Accounts構造体内で、権限を要求するアカウント（例: Escrow, Vaultなど）に対し、適切な`has_one`や`constraint`属性が設定されていることを確認する。
- [ ] `Signer<'info>`がAccounts構造体に含まれているか確認する。

**確認例:**

```rust
#[derive(Accounts)]
pub struct SecureAuthorization<'info> {
    pub authority: Signer<'info>,
    #[account(mut, has_one = authority)]  // authorityチェックが明示的に指定されている
    pub escrow: Account<'info, Escrow>,
}

### ✔️ Step 2: ロジック内の権限者検証（Anchor・非Anchor両方）

- [ ] 各インストラクション関数を確認し、署名者（`authority.key()`）と対象アカウント内の権限フィールド（`account.authority`）が明示的に比較されていることを確認する。
- [ ] 権限チェックが行われないまま重要なフィールドが変更される箇所がないことを確認する。

**危険なコード例:**

```rust
// ❌ 署名者の検証がない
pub fn insecure_authorization(ctx: Context<InsecureAuthorization>, data: u8) -> Result<()> {
    ctx.accounts.escrow.data = data;
    Ok(())
}

**安全なコード例:**

```rust

// ✅ 署名者を明示的に検証
pub fn secure_authorization(ctx: Context<SecureAuthorization>, data: u8) -> Result<()> {
    require!(ctx.accounts.escrow.authority == ctx.accounts.authority.key(), Unauthorized);
    ctx.accounts.escrow.data = data;
    Ok(())
}

### ✔️ Step 3: PDAに関する権限チェック

- [ ] PDAをAuthorityとして使用する場合、PDAの署名（`signer_seeds`）を適切に行い、署名者との関連付けが明示的に行われていることを確認する。
- [ ] PDA内の`authority`フィールドが署名者と一致することがコードで明示的に検証されているか確認する。

**確認例（PDAの場合）:**

```rust
let seeds = &[b"escrow", authority.key().as_ref(), &[bump]];
let signer = &[&seeds[..]];
invoke_signed(&instruction, accounts, signer)?;

## 📂 確認するコード箇所（具体的な確認対象）

- 各インストラクション関数の冒頭部
- `#[derive(Accounts)]`構造体内の各アカウント定義
- PDAを生成または利用しているすべての箇所
- 特にデータ更新やSOLの移動、トークンの転送などの重要な操作を行う箇所

---

## 💬 監査中に確認する質問例

- 「このインストラクションを呼び出せる署名者は誰か？」
- 「アカウント内の権限フィールド（authorityなど）は、署名者と明確に関連付けられているか？」
- 「この操作を意図しない署名者が実行できる可能性はないか？」

---

## 🚨 リスク評価基準

監査中に以下を検出した場合は重大な問題として報告する:

- 署名者の検証がまったく存在しない
- 権限をもつアカウントが署名者と関連付けられていない
- PDAの署名検証が不足しており、悪意ある署名が可能となっている

---

## 🛠 推奨する修正方法

- **Anchor使用の場合:**
  - `has_one`や`constraint`属性を追加する。

- **Anchor未使用の場合:**
  - 明示的なチェック（`require!`や`assert_eq!`）をコードに追加する。

- **PDAを使用する場合:**
  - `signer_seeds`を必ず使い、authorityフィールドとの比較を追加する。

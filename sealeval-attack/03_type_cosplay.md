# Type Cosplay（型の混同 / 型偽装）

## 概要

Solanaのアカウントは単なるバイト列として保存されており、プログラム側で期待する型にデシリアライズ（構造体に変換）します。型チェックを怠ると、同じサイズの別構造体に誤ってデシリアライズされてもエラーが出ない場合があります。

例えばサイズが同一のUser構造体とUserMetadata構造体があったとします。本来User型のアカウントを期待していても、攻撃者が中身が異なるUserMetadata型のアカウントを差し出してもサイズが合えばデシリアライズが通ってしまい、プログラムが誤って受け入れてしまう可能性があります。これを「Type Cosplay（型のなりすまし）」攻撃といいます。

## 対処法

アカウントに識別子（ディスクリミネータ）を設けて型を判別することが重要です。Anchorでは`#[account]`属性を付けた構造体に自動的に8バイトのディスクリミネータが付き、デシリアライズ時にこの識別子で型チェックを行います。

Anchorを使わない場合でも、自前で構造体の先頭に種別を示すフィールドを入れてチェックする方法があります（例えば列挙型でUserかMetadataかを持たせる等）。また、AnchorのAccount<'info, Type>を使えば、AccountInfoから特定の型に変換する際にAnchor runtimeが自動でディスクリミネータを確認するため、安全です。

要するにアカウントの型を厳格に特定することで、型偽装を防ぎます。

## 危険なコード例

以下のコードでは、account_infoを直接User型としてパースしています。しかしUserと同サイズの別構造を渡された場合、try_from_sliceはエラーを出さずにデータを読み込んでしまう恐れがあります。

```rust
// 不正: AccountInfoを直接Userとして解析。型識別をしていない。
pub fn insecure_user_read(ctx: Context<InsecureTypeCosplay>) -> Result<()> {
    let user: User = User::try_from_slice(&ctx.accounts.user.data.borrow())?;  
    // ※UserMetadata型のデータでもバイト長が同じなら通ってしまう可能性
    msg!("User age: {}", user.age);
    Ok(())
}

#[derive(Accounts)]
pub struct InsecureTypeCosplay<'info> {
    /// CHECK: 型チェックを故意に外している
    pub user: AccountInfo<'info>,  // 任意のアカウントを渡せてしまう
    pub authority: Signer<'info>,
}
```

## 安全なコード例

Anchorを使う場合、初期化時に8バイトのディスクリミネータが自動付与されます。また`Account<'info, User>`型で受け取ればAnchorが内部で型チェックをします。さらに`has_one`などで他の関連も確認できます。

```rust
#[account]  // Anchorにより8バイトの識別子が付与される
pub struct User { 
    pub authority: Pubkey,
    // ... その他フィールド
}

#[derive(Accounts)]
pub struct SecureTypeCosplay<'info> {
    #[account(has_one = authority)]
    pub user: Account<'info, User>,   // User型として受け取り、Anchorが型整合性をチェック
    pub authority: Signer<'info>,
}
```

上記では、`Account<'info, User>`型を使うことでuserアカウントがUser構造体として正しいものか（ディスクリミネータ一致するか）自動検証されます。結果、サイズが同じ別構造体を渡す型偽装攻撃を防ぐことができます。

## 追加情報

型の偽装攻撃は特に以下のような状況で発生しやすいです：

1. 複数の似た構造体を持つプログラム
2. 手動でデシリアライズを行うコード
3. 型チェックを省略したカスタム検証ロジック

Anchorを使用する場合の型安全性確保のポイント：

- 常に`#[account]`属性を使用してアカウント構造体を定義する
- `AccountInfo`の代わりに`Account<'info, Type>`を使用する
- 手動でデシリアライズする場合は、最初に8バイトのディスクリミネータを検証する

Anchorを使用しない場合でも、以下の対策が有効です：

- 構造体の先頭に型識別子フィールドを追加する
- デシリアライズ前に識別子を検証する
- 可能な限り型安全なデシリアライズ方法を使用する

型の偽装攻撃は、一見すると気づきにくい脆弱性ですが、適切な型チェックを実装することで効果的に防止できます。


# 📝 監査手順書: Type Cosplay（型の混同 / 型偽装）

---

## 📌 監査目的

- プログラム内で使用されるアカウントデータが、期待する型（構造体）であることを確実に保証する。
- サイズやデータ構造が似た別の型を不正に渡され、型を偽装した攻撃を防止する。

---

## 🔎 監査観点

監査者は以下の観点でコードを精査する。

### 1. Anchor使用時の型チェック確認

- Anchorの`#[account]`属性を使用しているか。
- Anchorの`Account<'info, T>`を使用し、型が明示的に指定されているか。

### 2. Anchor未使用時の型チェック確認

- 自前の型識別子（ディスクリミネータ）をデータの先頭に設け、デシリアライズ前に識別子をチェックしているか。

### 3. デシリアライズ手順確認

- `AccountInfo`型を使う場合、`try_from_slice`などの手動デシリアライズ前に明示的に型の整合性をチェックしているか。

---

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Accounts構造体の型チェック確認（Anchor使用時）

- [ ] アカウント構造体定義に`#[account]`属性が適切に付与されているか確認する。
- [ ] Accounts構造体内で、アカウントを`Account<'info, T>`として指定し、Anchorによる型チェックが確実に行われるようになっているか確認する。

**良い例:**

```rust
#[derive(Accounts)]
pub struct UserOperation<'info> {
    #[account(has_one = authority)]
    pub user: Account<'info, User>,
    pub authority: Signer<'info>,
}

#[account]
pub struct User {
    pub authority: Pubkey,
    pub data: u64,
}
```

### ✔️ Step 2: 手動デシリアライズ時の型識別確認（Anchor未使用時）

- [ ] 手動で`try_from_slice`や`deserialize`を行う場合、構造体の先頭に型識別子を置き、デシリアライズ前に明示的にチェックしているか確認する。

**良い例（Anchor未使用時）:**

```rust
let data = account_info.data.borrow();
let (discriminator, rest) = data.split_at(8);
if discriminator != USER_DISCRIMINATOR {
    return Err(ErrorCode::InvalidAccountType.into());
}
let user: User = User::try_from_slice(rest)?;
```

### ✔️ Step 3: 類似した構造体が存在する場合の検証

- [ ] サイズやデータ構造が類似している複数のアカウント構造体が存在する場合、型識別のロジックが各型に明示的に用意されているか確認する。

---

## 📂 確認するコード箇所（具体的な確認対象）

- Anchorの`#[account]`属性付きのアカウント構造体定義箇所
- Accounts構造体内でのアカウント型指定箇所（`Account<'info, T>`）
- 手動で`AccountInfo`をデシリアライズしている関数の全箇所
- 型識別のためのディスクリミネータ（8バイト）が定義されている箇所

---

## 💬 監査中に確認する質問例

- 「このアカウントのデータが想定通りの型であることをどのように保証していますか？」
- 「型偽装（Type Cosplay）攻撃を防止するためのディスクリミネータなどの仕組みがありますか？」
- 「サイズや構造が似ている複数のアカウント型を識別する仕組みがありますか？」

---

## 🚨 リスク評価基準

監査中に以下を検出した場合は重大な問題として報告する:

- `AccountInfo`型のデータを直接デシリアライズし、型識別をまったく行っていない
- Anchorを使用しているにもかかわらず、アカウント構造体に`#[account]`属性がなく、自動型チェックが働かない状態である
- サイズや構造が類似したアカウントが存在するのに、型を識別する仕組みが一切存在しない

---

## 🛠 推奨する修正方法

- **Anchor使用の場合:**
  - アカウント構造体に`#[account]`属性を追加する
  - `AccountInfo`の代わりに`Account<'info, T>`を使用する

- **Anchor未使用の場合:**
  - 構造体の先頭に型識別子フィールドを追加する
  - デシリアライズ前に識別子を検証するコードを追加する

**修正後の推奨コード例（Anchor使用）:**

```rust
#[account]
pub struct User {
    pub authority: Pubkey,
    pub data: u64,
}

#[derive(Accounts)]
pub struct UserOperation<'info> {
    #[account(has_one = authority)]
    pub user: Account<'info, User>,
    pub authority: Signer<'info>,
}

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

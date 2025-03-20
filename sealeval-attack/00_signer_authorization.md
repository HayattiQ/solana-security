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

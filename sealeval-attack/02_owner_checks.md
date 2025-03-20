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

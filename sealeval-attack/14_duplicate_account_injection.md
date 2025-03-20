# Duplicate Account Injection（重複アカウント注入攻撃）

## 概要

複数のアカウント引数を受け取る際、攻撃者が意図的に同一アカウントを複数の引数として渡すことで、本来別々に扱うべき処理が同一アカウントに対して二重に行われる可能性があります。

既に説明した「Duplicate Mutable Accounts」と似ていますが、より広義に引数全体の重複チェックとして注意すべきケースです。

## 対処法

1. 関数冒頭で、各アカウントのPubkeyを比較し、重複がないか確認する
2. Anchorでは、アカウント構造体の`constraint`属性を用いて、明示的に重複しないことを保証する

## 危険なコード例

以下の例では、複数の引数として受け取るアカウント間の重複チェックをしていません：

```rust
// 不正: 複数の引数として受け取るアカウント間の重複チェックをしていない
pub fn insecure_process_accounts(ctx: Context<InsecureAccounts>) -> Result<()> {
    // 攻撃者が同一アカウントを渡しても検証されない
    ctx.accounts.account_a.balance += 10;
    ctx.accounts.account_b.balance -= 10;
    Ok(())
}
```

## 安全なコード例

以下の例では、アカウント間の重複をチェックしています：

```rust
pub fn secure_process_accounts(ctx: Context<SecureAccounts>) -> Result<()> {
    require!(ctx.accounts.account_a.key() != ctx.accounts.account_b.key(), ProgramError::InvalidArgument);
    ctx.accounts.account_a.balance += 10;
    ctx.accounts.account_b.balance -= 10;
    Ok(())
}

#[derive(Accounts)]
pub struct SecureAccounts<'info> {
    #[account(mut)]
    pub account_a: Account<'info, BalanceAccount>,
    #[account(mut, constraint = account_a.key() != account_b.key())]
    pub account_b: Account<'info, BalanceAccount>,
}
```

## 追加情報

### 重複アカウント注入の危険性

重複アカウント注入攻撃は、特に以下のような状況で危険です：

1. **資金移動操作**
   - 送金元と送金先が同一アカウントの場合、残高計算が不正確になる可能性があります
   - 例：A→Bへの送金で、AとBに同じアカウントを指定すると、残高が変わらない可能性があります

2. **権限チェックのバイパス**
   - 権限を持つアカウントと検証対象アカウントが同一の場合、検証が無効化される可能性があります
   - 例：所有者アカウントとトークンアカウントに同じアカウントを指定すると、所有権チェックが無効になる可能性があります

3. **状態の不整合**
   - 同一アカウントに対する複数の操作が、予期しない順序で実行される可能性があります
   - 例：更新と検証が同一アカウントに対して行われると、検証が無効になる可能性があります

### 包括的な重複チェック戦略

1. **すべてのアカウントペアの検証**

   多数のアカウントを扱う場合、すべての組み合わせをチェックする必要があります：

   ```rust
   // 3つのアカウントの場合、3つの組み合わせをチェック
   require!(account_a.key() != account_b.key(), ProgramError::InvalidArgument);
   require!(account_a.key() != account_c.key(), ProgramError::InvalidArgument);
   require!(account_b.key() != account_c.key(), ProgramError::InvalidArgument);
   ```

   Anchorでは、これを以下のように宣言的に表現できます：

   ```rust
   #[derive(Accounts)]
   pub struct MultipleAccounts<'info> {
       #[account(mut)]
       pub account_a: Account<'info, SomeType>,
       
       #[account(mut, constraint = account_a.key() != account_b.key())]
       pub account_b: Account<'info, SomeType>,
       
       #[account(
           mut, 
           constraint = account_a.key() != account_c.key() && account_b.key() != account_c.key()
       )]
       pub account_c: Account<'info, SomeType>,
   }
   ```

2. **関数型アプローチ**

   多数のアカウントがある場合、ヘルパー関数を使用して重複チェックを行うことができます：

   ```rust
   fn check_no_duplicate_accounts(accounts: &[AccountInfo]) -> Result<()> {
       for i in 0..accounts.len() {
           for j in i+1..accounts.len() {
               if accounts[i].key == accounts[j].key {
                   return Err(ProgramError::InvalidArgument.into());
               }
           }
       }
       Ok(())
   }
   
   // 使用例
   pub fn process(ctx: Context<ProcessAccounts>) -> Result<()> {
       let accounts = [
           ctx.accounts.account_a.to_account_info(),
           ctx.accounts.account_b.to_account_info(),
           ctx.accounts.account_c.to_account_info(),
       ];
       check_no_duplicate_accounts(&accounts)?;
       
       // 安全に処理を続行...
       Ok(())
   }
   ```

3. **アカウント設計の見直し**

   重複アカウントの問題が発生しやすい場合は、アカウント構造の設計を見直すことも検討すべきです：

   - 明確な役割分担：各アカウントの役割を明確に分け、混同しにくい設計にする
   - 型の区別：異なる役割には異なる型を使用し、型システムで区別できるようにする
   - 関連性の明示：アカウント間の関連性を明示的に設計し、検証しやすくする

重複アカウント注入攻撃は、単純なチェックで防止できますが、見落としやすい脆弱性です。特に複数のアカウントを操作する関数では、常に重複チェックを実装することを習慣づけましょう。

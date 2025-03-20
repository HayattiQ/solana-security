# Duplicate Mutable Accounts（一つのアカウントを重複利用する攻撃）

## 概要

関数の引数で同じ型のアカウントを複数可変（mutable）で受け取る場合、攻撃者は同一のアカウントを2つの引数に渡すことで不正を仕掛ける可能性があります。

本来別々であるはずのアカウントが同一だった場合、プログラム内の処理ロジックが意図しない結果を生みます。例えば2つのユーザーバランスを操作して送金する関数で、両方に同じアカウントが渡されたら、片方の処理がもう片方にも影響してしまい整合性が崩れるといった事態が起こります。

## 対処法

引数で受け取った複数アカウントが同一でないことをチェックします。単純ですが強力な防御策です。Anchorでは`#[account(constraint = accountA.key() != accountB.key())]`という書き方で2つのアカウントのPubkey不一致を検証できます。

同一だった場合はエラーを出し、処理を中断します。また、根本的に同じアカウントを2度渡す必要がない設計であれば、そもそもそういう呼び出しを許さないのが望ましいです。

## 危険なコード例

2つのボールト間で資産を移動する関数で、vault_aとvault_bを引数に取っているとします。検証がないと、攻撃者は両方に同じvaultを指定できます。

```rust
// 不正: vault_aとvault_bが同じ場合の処理を考慮していない
pub fn atomic_swap(ctx: Context<AtomicSwap>, amount: u64) -> Result<()> {
    let vault_a = &mut ctx.accounts.vault_a;
    let vault_b = &mut ctx.accounts.vault_b;
    // それぞれに対して金額を加減算する処理
    vault_a.balance += amount;
    vault_b.balance -= amount;
    Ok(())
}
```

上記では、もしvault_aとvault_bが同一アカウントなら、残高計算が二重に適用され不正な値になります。

## 安全なコード例

関数内またはAccounts検証で、`vault_a.key() != vault_b.key()`をチェックします。

```rust
pub fn atomic_swap_secure(ctx: Context<AtomicSwap>, amount: u64) -> Result<()> {
    let vault_a = &mut ctx.accounts.vault_a;
    let vault_b = &mut ctx.accounts.vault_b;
    require!(vault_a.key() != vault_b.key(), SwapError::SameAccount);
    // 安全: ここから下はvault_aとvault_bは別アカウントであることが保証される
    vault_a.balance += amount;
    vault_b.balance -= amount;
    Ok(())
}
```

Anchorでは次のようにアカウント構造体で宣言可能です。

```rust
#[derive(Accounts)]
pub struct AtomicSwap<'info> {
    #[account(mut)]
    pub vault_a: Account<'info, Vault>,
    #[account(mut, constraint = vault_a.key() != vault_b.key())]  // 異なることを保証
    pub vault_b: Account<'info, Vault>,
    // ...省略...
}
```

このようにすることで、同じアカウントを二重に渡す攻撃をシンプルに防止できます。

## 追加情報

重複アカウント攻撃は、特に以下のような状況で発生しやすいです：

1. 送金や交換など、複数のアカウント間で値を移動させる操作
2. 複数の同じ型のアカウントを引数に取る関数
3. 複雑な計算や状態変更を行うトランザクション

### 重複アカウント検証のベストプラクティス

1. **すべての可変アカウントペアを検証する**

   複数の可変アカウントがある場合、すべての組み合わせで重複がないことを確認します。例えば3つのアカウントa, b, cがある場合、a≠b, a≠c, b≠cの3つの検証が必要です。

   ```rust
   #[derive(Accounts)]
   pub struct MultiAccountOperation<'info> {
       #[account(mut)]
       pub account_a: Account<'info, SomeType>,
       #[account(mut, constraint = account_a.key() != account_b.key())]
       pub account_b: Account<'info, SomeType>,
       #[account(mut, constraint = account_a.key() != account_c.key() && account_b.key() != account_c.key())]
       pub account_c: Account<'info, SomeType>,
   }
   ```

2. **関数内での追加検証**

   Anchorのアカウント検証に加えて、関数内でも重要な操作の前に再度チェックすることで、二重の安全性を確保できます。

   ```rust
   pub fn process_accounts(ctx: Context<MultiAccountOperation>) -> Result<()> {
       // 念のため再検証
       let a = &ctx.accounts.account_a;
       let b = &ctx.accounts.account_b;
       let c = &ctx.accounts.account_c;
       
       if a.key() == b.key() || a.key() == c.key() || b.key() == c.key() {
           return Err(ProgramError::InvalidArgument.into());
       }
       
       // 安全な処理を続行...
       Ok(())
   }
   ```

3. **設計レベルでの対策**

   可能であれば、同じ型のアカウントを複数受け取る設計自体を見直すことも検討します。例えば、送金元と送金先を明確に区別できる異なる型や構造を使用することで、混同のリスクを減らせます。

重複アカウント攻撃は単純な検証で防げるため、複数の可変アカウントを扱う際は必ず重複チェックを実装しましょう。

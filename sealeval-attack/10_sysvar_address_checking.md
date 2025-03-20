# Sysvar Address Checking（Sysvarアドレスの検証不足）

## 概要

SolanaにはClockやRentなどのSysvar（システム変数）アカウントがあります。これらは決まったPubkeyでネットワーク上に存在しますが、プログラムの引数として要求する際に正しいSysvarアドレスであることを確認しないと、攻撃者が偽のSysvar風アカウントを渡す可能性があります。

Sysvarアカウントは通常readonlyでシステムプログラムが所有していますが、「所有者チェック済みだから安心」と思い込み、アドレス自体を確認しないと危険です。例えば、本物のRent Sysvarではない任意のAccountInfoをRent Sysvarとして渡されても、プログラムがそれを信じてしまうと誤った状態データを読み込む可能性があります。

## 対処法

SysvarアカウントのPubkeyが期待する定数と一致することを必ずチェックします。Anchorでは、`Sysvar<'info, Rent>`のような型を使えば自動的に正しいアドレスであることを保証します（Anchorは内部でそのPubkeyをSolanaの定数と比較します）。

もし`AccountInfo`で受け取る場合でも、関数内で例えば`require_eq!(rent_account.key(), sysvar::rent::ID)`のようにして、与えられたrent_accountのPubkeyがSolanaのRent Sysvarのアドレスと一致するか確認します。

またSolanaプログラムSDKには`Rent::get()`や`Clock::get()`といったヘルパーがあり、これを使えばプログラム内でわざわざ引数に取らずとも最新のSysvar情報を取得できます（より安全な方法です）。

総じて、Sysvarはアドレスで認証することで、攻撃者の偽装を防ぎます。

## 危険なコード例

SysvarアカウントをAccountInfoで受け取り、そのまま利用している例です。

```rust
pub fn use_sysvars(ctx: Context<UseSysvars>) -> Result<()> {
    let rent_info = &ctx.accounts.rent;  // Sysvar: Rentを期待
    msg!("Rent account key: {}", rent_info.key());  // 単に表示するだけだが…
    // もしrent_infoがRent Sysvarでなかった場合、本来ありえない挙動をする可能性
    Ok(())
}

#[derive(Accounts)]
pub struct UseSysvars<'info> {
    /// CHECK: 検証しないと危険
    pub rent: AccountInfo<'info>,  // 本当はSysvar<'info, Rent>を使うべき
}
```

上記では特に何もしていませんが、例えばRent情報を参照して計算するような場合、攻撃者が細工した偽Rentデータを持つアカウントを渡せてしまいます。

## 安全なコード例

Anchorではシンプルに以下のように書けます。

```rust
#[derive(Accounts)]
pub struct UseSysvarsSecure<'info> {
    pub rent: Sysvar<'info, Rent>,  // AnchorがRentのアドレス(SysvarRent111111111111111111111111111111111)を保証
}

pub fn use_sysvars_secure(ctx: Context<UseSysvarsSecure>) -> Result<()> {
    let rent = ctx.accounts.rent.clone();  // 安全にRent sysvarデータが取得できる
    msg!("Rent minimum balance: {}", rent.minimum_balance(0));
    Ok(())
}
```

もし`AccountInfo`で受け取る必要がある特殊な場合でも、関数内で次のようにチェックします。

```rust
if ctx.accounts.rent.key() != solana_program::sysvar::rent::ID {
    return Err(MyError::InvalidSysvar.into());
}
```

これで、与えられたアカウントがRent Sysvarのアドレスと一致しているか確認できます。Clockなど他のSysvarでも同様です。以上により、Sysvarを悪用した偽装攻撃を防止できます。

## 追加情報

### Solanaの主なSysvar

Solanaには以下のような主要なSysvarがあります：

1. **Clock** (`solana_program::sysvar::clock::ID`)
   - 現在のスロット、エポック、Unix時間などの時間情報を提供
   - アドレス: `SysvarC1ock11111111111111111111111111111111`

2. **Rent** (`solana_program::sysvar::rent::ID`)
   - アカウントのレント（賃料）計算に関する情報を提供
   - アドレス: `SysvarRent111111111111111111111111111111111`

3. **EpochSchedule** (`solana_program::sysvar::epoch_schedule::ID`)
   - エポックのスケジュール情報を提供
   - アドレス: `SysvarEpochSchedu1e111111111111111111111111`

4. **Fees** (`solana_program::sysvar::fees::ID`)
   - 現在のトランザクション手数料に関する情報を提供
   - アドレス: `SysvarFees111111111111111111111111111111111`

5. **SlotHashes** (`solana_program::sysvar::slot_hashes::ID`)
   - 最近のスロットとそのハッシュ値のリストを提供
   - アドレス: `SysvarS1otHashes111111111111111111111111111`

6. **SlotHistory** (`solana_program::sysvar::slot_history::ID`)
   - 最近のスロットの履歴を提供
   - アドレス: `SysvarS1otHistory11111111111111111111111111`

### Sysvarへのアクセス方法

Sysvarにアクセスする方法は主に3つあります：

1. **直接アクセス（推奨）**
   
   Solana SDKの`get()`メソッドを使用して直接Sysvarにアクセスする方法です。これが最も安全で簡潔な方法です。
   
   ```rust
   let clock = Clock::get()?;
   let current_slot = clock.slot;
   
   let rent = Rent::get()?;
   let lamports_required = rent.minimum_balance(data_size);
   ```

2. **Anchorの型を使用**
   
   Anchorを使用している場合、`Sysvar<'info, T>`型を使用してSysvarを安全に取得できます。
   
   ```rust
   #[derive(Accounts)]
   pub struct MyContext<'info> {
       pub clock: Sysvar<'info, Clock>,
       pub rent: Sysvar<'info, Rent>,
   }
   ```

3. **AccountInfoで受け取って検証**
   
   特殊な理由でAccountInfoとして受け取る必要がある場合は、必ずアドレスを検証します。
   
   ```rust
   #[derive(Accounts)]
   pub struct MyContext<'info> {
       /// CHECK: We verify this is the clock sysvar in the instruction
       pub clock: AccountInfo<'info>,
   }
   
   pub fn my_instruction(ctx: Context<MyContext>) -> Result<()> {
       // アドレスを検証
       if ctx.accounts.clock.key() != solana_program::sysvar::clock::ID {
           return Err(ProgramError::InvalidArgument.into());
       }
       
       // 安全に使用
       let clock = Clock::from_account_info(&ctx.accounts.clock)?;
       let current_slot = clock.slot;
       
       Ok(())
   }
   ```

### セキュリティのベストプラクティス

1. **可能な限り直接アクセスを使用する**
   - `Clock::get()`や`Rent::get()`などの直接アクセス方法を優先的に使用する
   - これにより、Sysvarアカウントを引数として渡す必要がなくなり、攻撃ベクトルを減らせる

2. **Anchorの型安全機能を活用する**
   - Anchorを使用する場合は、`Sysvar<'info, T>`型を使用してSysvarを安全に取得する
   - これにより、アドレス検証が自動的に行われる

3. **常にアドレスを検証する**
   - AccountInfoとしてSysvarを受け取る場合は、必ずアドレスを検証する
   - 検証前にSysvarのデータにアクセスしない

Sysvarアドレスの検証は、Solanaプログラムのセキュリティにおいて見落とされがちな重要な要素です。適切な検証を実装することで、偽のSysvarアカウントによる攻撃を効果的に防止できます。


# 📝 監査手順書: Sysvar Address Checking（Sysvarアドレスの検証不足）

## 📌 監査目的
- Sysvar（システム変数）アカウントが正しいアドレスであることを確認する。
- 偽造されたSysvarアカウントを利用した攻撃を防止する。

## 🔎 監査観点
監査者は以下の観点でコードを精査すること。

1. **Sysvarアドレスの検証**
   - Sysvarを利用する場合、必ずアドレスがSolana公式の定数（例: `sysvar::rent::ID`）と一致していることを確認する。

2. **Anchor利用時のSysvar型安全性**
   - Anchorを使用する場合、Sysvarアカウントは`Sysvar<'info, T>`型で宣言されているか確認する。

3. **直接アクセスメソッドの利用**
   - 可能な限り、`Rent::get()` や `Clock::get()` 等の直接アクセス方法を使用していることを確認する。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Anchorを使用したSysvar型の確認
- `[ ]` Anchorを使用している場合、Sysvarアカウントは`Sysvar<'info, T>`型で宣言されていることを確認する。

**安全な例（推奨）:**
```rust
#[derive(Accounts)]
pub struct MyContext<'info> {
    pub rent: Sysvar<'info, Rent>,
    pub clock: Sysvar<'info, Clock>,
}
```

### ✔️ Step 2: AccountInfo利用時の明示的なアドレス検証確認
- `[ ]` 特殊な理由で`AccountInfo`としてSysvarを取得する場合、必ず明示的にアドレスが検証されていることを確認する。

**安全な例:**
```rust
pub fn my_instruction(ctx: Context<MyContext>) -> Result<()> {
    if ctx.accounts.rent.key() != solana_program::sysvar::rent::ID {
        return Err(ProgramError::InvalidArgument.into());
    }

    let rent = Rent::from_account_info(&ctx.accounts.rent)?;
    Ok(())
}
```

### ✔️ Step 3: 直接アクセスメソッドの推奨使用確認
- `[ ]` Sysvarを引数として渡さず、プログラム内で直接`Clock::get()`や`Rent::get()`を使用している場合、その方法が適切に実装されていることを確認する。

**安全な例（推奨）:**
```rust
let clock = Clock::get()?;
let rent = Rent::get()?;
```

## 📂 確認するコード箇所（具体的な確認対象）
- AnchorのAccounts構造体内のSysvar宣言箇所
- AccountInfoとしてSysvarを受け取っている全ての関数
- Sysvarに直接アクセスしている関数（`Rent::get()`、`Clock::get()` などの使用箇所）

## 💬 監査中に確認する質問例
- 「Sysvarアカウントのアドレスが明示的に検証されていますか？」
- 「Anchorを利用している場合、Sysvarアカウントは`Sysvar<'info, T>`型を利用していますか？」
- 「Sysvarを直接プログラム内で取得する方法（`Clock::get()`等）は適切に使用されていますか？」

## 🚨 リスク評価基準

**重大な問題として報告すべき場合:**
- `AccountInfo`型のSysvarアカウントに対して、アドレス検証が全く行われていない場合（攻撃者が偽造Sysvarを渡せる可能性がある）。

**改善推奨として報告すべき場合:**
- Anchorを使用しているにもかかわらず、Sysvarアカウントが`AccountInfo`で受け取られている場合（型安全性向上のため`Sysvar<'info, T>`型を推奨）。
- 不必要にSysvarアカウントを引数として渡している場合（`Rent::get()`や`Clock::get()`の直接アクセスを推奨）。

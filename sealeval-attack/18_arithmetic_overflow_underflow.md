# 算術オーバーフロー/アンダーフロー

## 概要

Rustは多くの場合安全ですが、リリースビルドなどでデフォルトのチェックが無効になっている場合、算術演算のオーバーフローやアンダーフローが発生し、予期しない動作や数値のラップアラウンドにつながる可能性があります。

## 対処法

1. 必要に応じて`checked_add`、`checked_sub`、`checked_mul`などの算術演算関数を利用する
2. アサーションや境界チェックを明示的に追加し、異常値が入力された場合はエラーを返す

## 危険なコード例

以下の例では、標準の加算を利用しており、リリースモードでオーバーフローが起こる可能性があります：

```rust
// 不正: 標準の加算を利用し、リリースモードでオーバーフローが起こる可能性
let new_balance = account.balance + deposit_amount;
```

## 安全なコード例

以下の例では、`checked_add`を用いてオーバーフロー時にエラーを返しています：

```rust
// 安全: checked_add を用い、オーバーフロー時はエラーを返す
let new_balance = account.balance.checked_add(deposit_amount)
    .ok_or(ProgramError::InvalidArgument)?;
```

## 追加情報

### 算術オーバーフロー/アンダーフローの危険性

算術オーバーフロー/アンダーフローは、特に金融アプリケーションにおいて深刻な問題を引き起こす可能性があります：

1. **残高の不正操作**
   - オーバーフローにより、大きな数値が小さな数値になる可能性があります
   - アンダーフローにより、負の数値が大きな正の数値になる可能性があります

2. **ビジネスロジックの破壊**
   - 計算結果が予期しない値になることで、条件分岐や比較が誤った結果を返す可能性があります
   - これにより、本来通過すべきでないチェックが通過してしまう可能性があります

3. **セキュリティ境界の突破**
   - 攻撃者が意図的にオーバーフロー/アンダーフローを引き起こし、セキュリティチェックをバイパスする可能性があります

### 安全な算術演算のパターン

Rustでは、以下のような安全な算術演算関数が提供されています：

1. **チェック付き演算**

   オーバーフロー/アンダーフローが発生した場合に`None`を返します：

   ```rust
   // 加算
   let result = a.checked_add(b).ok_or(MyError::Overflow)?;
   
   // 減算
   let result = a.checked_sub(b).ok_or(MyError::Underflow)?;
   
   // 乗算
   let result = a.checked_mul(b).ok_or(MyError::Overflow)?;
   
   // 除算
   let result = a.checked_div(b).ok_or(MyError::DivisionByZero)?;
   ```

2. **飽和演算**

   オーバーフロー/アンダーフローが発生した場合に型の最大値/最小値で飽和します：

   ```rust
   // 加算（オーバーフローすると最大値になる）
   let result = a.saturating_add(b);
   
   // 減算（アンダーフローすると最小値（0または負の最大値）になる）
   let result = a.saturating_sub(b);
   ```

3. **ラッピング演算**

   オーバーフロー/アンダーフローが発生した場合に明示的にラップアラウンドします：

   ```rust
   // 加算（オーバーフローすると0からやり直す）
   let result = a.wrapping_add(b);
   
   // 減算（アンダーフローすると最大値からやり直す）
   let result = a.wrapping_sub(b);
   ```

4. **オーバーフロー情報付き演算**

   オーバーフロー/アンダーフローが発生したかどうかの情報も返します：

   ```rust
   // 加算（結果とオーバーフローフラグのタプルを返す）
   let (result, overflowed) = a.overflowing_add(b);
   if overflowed {
       return Err(MyError::Overflow);
   }
   ```

### Solanaプログラムでの実装例

Solanaプログラムでは、特に以下のような場面で安全な算術演算が重要です：

1. **トークン残高の操作**

   ```rust
   // 安全なトークン転送
   pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
       let from = &mut ctx.accounts.from;
       let to = &mut ctx.accounts.to;
       
       // 送金元の残高チェック
       require!(from.balance >= amount, MyError::InsufficientFunds);
       
       // 安全な減算
       from.balance = from.balance.checked_sub(amount)
           .ok_or(MyError::Underflow)?;
       
       // 安全な加算
       to.balance = to.balance.checked_add(amount)
           .ok_or(MyError::Overflow)?;
       
       Ok(())
   }
   ```

2. **手数料計算**

   ```rust
   // 安全な手数料計算
   pub fn calculate_fee(amount: u64, fee_rate: u64) -> Result<u64> {
       // 手数料計算（amount * fee_rate / 10000）
       let fee = amount.checked_mul(fee_rate)
           .ok_or(MyError::Overflow)?
           .checked_div(10000)
           .ok_or(MyError::DivisionByZero)?;
       
       Ok(fee)
   }
   ```

3. **複合計算**

   ```rust
   // 複数の演算を組み合わせる場合
   pub fn complex_calculation(a: u64, b: u64, c: u64) -> Result<u64> {
       // (a + b) * c
       let sum = a.checked_add(b)
           .ok_or(MyError::Overflow)?;
       
       let result = sum.checked_mul(c)
           .ok_or(MyError::Overflow)?;
       
       Ok(result)
   }
   ```

算術オーバーフロー/アンダーフローは、適切なチェック関数を使用することで簡単に防止できます。特に金融関連の計算や重要な状態変更を行う場合は、必ず安全な算術演算関数を使用しましょう。

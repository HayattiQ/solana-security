# Immutable Data Bypass（不変データのバイパス攻撃）

## 概要

一部のアカウントは初期化時のみ設定され、その後変更されるべきでない「不変データ」として設計されることがあります。しかし、不十分な検証や更新可能なフィールドとして実装してしまうと、攻撃者が不変とすべきデータを書き換えることが可能となります。

## 対処法

1. 不変にすべきデータについては、初期化後の更新操作をプログラム上禁止するか、更新可能なフィールドと明確に分離する
2. Anchorでは、変更不可のフィールドについてのチェックや、`has_one`などで一度設定された値と照合する実装を行う
3. 関数内で更新処理を実行する前に、不変フラグ（例: `is_initialized`や`immutable`フィールド）を検証する

## 危険なコード例

以下の例では、初期化後も不変であるべきフィールドが自由に書き換えられてしまいます：

```rust
// 不正: 初期化後も不変であるべきフィールドが自由に書き換えられる
pub fn insecure_update(ctx: Context<InsecureUpdate>, new_value: u64) -> Result<()> {
    ctx.accounts.config.immutable_field = new_value; // 不変であるべきなのに更新可能
    Ok(())
}
```

## 安全なコード例

以下の例では、不変フィールドの更新を防止しています：

```rust
pub fn secure_update(ctx: Context<SecureUpdate>, new_value: u64) -> Result<()> {
    // immutable_fieldは初期化後変更不可として設計されているため、更新処理前にチェックする
    require!(ctx.accounts.config.immutable_field_locked, ProgramError::InvalidArgument);
    
    // immutable_field以外の更新可能なフィールドのみを変更する
    ctx.accounts.config.mutable_field = new_value;
    Ok(())
}

#[derive(Accounts)]
pub struct SecureUpdate<'info> {
    #[account(mut)]
    pub config: Account<'info, Config>,
}
```

※ 上記の例では、`immutable_field_locked`というフラグを初期化時に設定し、後からは変更できないように設計するなど、設計段階で不変性を担保する必要があります。

## 追加情報

### 不変データの重要性

不変データは、プログラムの信頼性とセキュリティにおいて重要な役割を果たします：

1. **契約の一貫性**
   - 一度設定された重要なパラメータ（手数料率、権限者など）が変更されないことで、プログラムの動作が予測可能になります
   - ユーザーは、これらの不変条件を信頼してプログラムと対話できます

2. **セキュリティの基盤**
   - 権限構造や重要な制約が不変であることで、攻撃ベクトルが制限されます
   - 例えば、プログラムの管理者や手数料受取先が不変であれば、これらを悪意ある値に変更する攻撃は不可能になります

3. **監査の容易さ**
   - 不変データは監査が容易で、プログラムの動作を理解しやすくなります
   - 変更可能な状態が少ないほど、プログラムの挙動を予測しやすくなります

### 不変データの実装パターン

1. **初期化時のみ設定可能なフィールド**

   ```rust
   #[account]
   pub struct Config {
       pub is_initialized: bool,
       pub admin: Pubkey,        // 初期化時のみ設定可能
       pub fee_rate: u64,        // 初期化時のみ設定可能
       pub mutable_setting: u64, // 更新可能
   }
   
   pub fn initialize(ctx: Context<Initialize>, fee_rate: u64) -> Result<()> {
       let config = &mut ctx.accounts.config;
       require!(!config.is_initialized, ProgramError::AccountAlreadyInitialized);
       
       config.is_initialized = true;
       config.admin = ctx.accounts.admin.key();
       config.fee_rate = fee_rate;
       config.mutable_setting = 0;
       
       Ok(())
   }
   
   pub fn update_setting(ctx: Context<UpdateSetting>, new_setting: u64) -> Result<()> {
       let config = &mut ctx.accounts.config;
       require!(config.is_initialized, ProgramError::UninitializedAccount);
       require!(config.admin == ctx.accounts.admin.key(), ProgramError::InvalidAuthority);
       
       // 不変フィールドは更新しない
       config.mutable_setting = new_setting;
       
       Ok(())
   }
   ```

2. **フィールドごとの更新ロック**

   ```rust
   #[account]
   pub struct AdvancedConfig {
       pub admin: Pubkey,
       pub fee_rate: u64,
       pub fee_rate_locked: bool,  // trueになると fee_rate は変更不可
       pub other_setting: u64,
   }
   
   pub fn lock_fee_rate(ctx: Context<AdminOnly>) -> Result<()> {
       let config = &mut ctx.accounts.config;
       require!(config.admin == ctx.accounts.admin.key(), ProgramError::InvalidAuthority);
       
       config.fee_rate_locked = true;  // 一度ロックすると解除不可
       
       Ok(())
   }
   
   pub fn update_fee_rate(ctx: Context<AdminOnly>, new_fee: u64) -> Result<()> {
       let config = &mut ctx.accounts.config;
       require!(config.admin == ctx.accounts.admin.key(), ProgramError::InvalidAuthority);
       require!(!config.fee_rate_locked, CustomError::FeeRateLocked);
       
       config.fee_rate = new_fee;
       
       Ok(())
   }
   ```

3. **Anchorの制約を活用**

   ```rust
   #[derive(Accounts)]
   pub struct UpdateConfig<'info> {
       #[account(
           mut,
           has_one = admin,
           constraint = !config.fee_rate_locked @ CustomError::FeeRateLocked
       )]
       pub config: Account<'info, Config>,
       pub admin: Signer<'info>,
   }
   ```

### 不変性の検証

不変データの保護には、複数の層での検証が効果的です：

1. **プログラム設計レベル**
   - 不変フィールドと可変フィールドを明確に分離する
   - 不変フィールドを更新する関数を提供しない

2. **アカウント構造体レベル**
   - 不変フィールドの状態を示すフラグを導入する
   - Anchorの制約を使用して不変条件を強制する

3. **関数レベル**
   - 更新操作の前に不変フィールドの状態を検証する
   - 不変フィールドへの変更を試みた場合はエラーを返す

不変データのバイパス攻撃は、適切な設計と検証によって効果的に防止できます。特に重要なパラメータや権限構造については、初期化後に変更できないよう慎重に設計することが重要です。

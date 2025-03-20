# CPI Data Injection Attack（CPI引数の偽装攻撃）

## 概要

Solanaでは、クロスプログラムインボケーション（CPI）を利用して他のプログラムを呼び出す際、引数データを攻撃者が操作できると、不正なデータが渡されて意図しない処理が実行される可能性があります。

特に、呼び出し先プログラムが引数の内容を前提として処理を行う場合、検証不足だと予期せぬ動作を招きます。

## 対処法

1. CPI呼び出し前に、渡す引数データの内容を十分に検証する
2. 可能であれば、Anchorのようなラッパーライブラリを用い、型安全な形で引数を構成する
3. CPI呼び出し時には、引数の正当性を担保するために内部検証（例えば、シリアライズ済みデータの整合性チェック）を行う

## 危険なコード例

以下の例では、CPI呼び出し時に引数の検証を省略しています：

```rust
// 不正: CPI呼び出し時に、引数の検証を省略している
pub fn insecure_cpi_call(ctx: Context<InsecureCPI>, amount: u64, extra_data: Vec<u8>) -> Result<()> {
    let cpi_accounts = SomeCpiAccounts {
        source: ctx.accounts.source.to_account_info(),
        destination: ctx.accounts.destination.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    };
    // extra_dataに不正な値が注入される可能性がある
    solana_program::program::invoke(
        &some_cpi_instruction(ctx.accounts.cpi_program.key, amount, extra_data),
        &[ctx.accounts.source.to_account_info(), ctx.accounts.destination.to_account_info()],
    )?;
    Ok(())
}
```

## 安全なコード例

以下の例では、CPI呼び出し前に引数データを十分に検証しています：

```rust
pub fn secure_cpi_call(ctx: Context<SecureCPI>, amount: u64, extra_data: Vec<u8>) -> Result<()> {
    // extra_dataの長さや値の範囲など、十分に検証する
    require!(extra_data.len() <= MAX_EXTRA_DATA_LEN, ProgramError::InvalidInstructionData);
    
    // CPIのアカウントや引数はAnchorの型安全な定義を利用
    let cpi_ctx = CpiContext::new(
        ctx.accounts.cpi_program.to_account_info(),
        SomeCpiAccounts {
            source: ctx.accounts.source.to_account_info(),
            destination: ctx.accounts.destination.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    );
    some_cpi_instruction::cpi::transfer(cpi_ctx, amount, extra_data)?;
    Ok(())
}
```

## 追加情報

### CPIセキュリティの重要性

クロスプログラムインボケーション（CPI）は、Solanaエコシステムの強力な機能ですが、適切に保護しないと重大なセキュリティリスクとなります。CPIを介した攻撃は特に危険です：

1. **権限の昇格**: 適切な検証なしにCPIを実行すると、攻撃者が本来持っていない権限を取得する可能性があります
2. **データの改ざん**: 不正な引数データにより、呼び出し先プログラムの動作を操作できる可能性があります
3. **意図しない副作用**: 検証されていないCPIは、予期しない状態変更を引き起こす可能性があります

### 安全なCPI実装のためのベストプラクティス

1. **引数の厳格な検証**
   
   すべてのCPI引数データは、呼び出し前に厳格に検証すべきです：
   
   ```rust
   // 数値の範囲チェック
   require!(amount > 0 && amount <= MAX_AMOUNT, InvalidAmount);
   
   // バイトデータの長さと内容チェック
   require!(data.len() <= MAX_DATA_LEN, DataTooLarge);
   if !data.is_empty() {
       // データの内容に応じた追加検証
       validate_data_format(&data)?;
   }
   ```

2. **Anchorの型安全なCPI**
   
   Anchorは型安全なCPI呼び出しを提供し、多くの一般的なエラーを防止します：
   
   ```rust
   // Anchorの型安全なCPI
   let cpi_ctx = CpiContext::new_with_signer(
       ctx.accounts.token_program.to_account_info(),
       token::Transfer {
           from: ctx.accounts.source.to_account_info(),
           to: ctx.accounts.destination.to_account_info(),
           authority: ctx.accounts.authority.to_account_info(),
       },
       &[&[seed, &[bump]]],
   );
   token::transfer(cpi_ctx, amount)?;
   ```

3. **呼び出し先プログラムの検証**
   
   CPI呼び出し先のプログラムIDが期待通りであることを確認します：
   
   ```rust
   // プログラムIDの検証
   require!(
       ctx.accounts.target_program.key() == expected_program_id,
       ProgramError::IncorrectProgramId
   );
   ```

4. **アカウント所有者の検証**
   
   CPI呼び出しに使用するアカウントの所有者を検証します：
   
   ```rust
   // アカウント所有者の検証
   require!(
       ctx.accounts.token_account.owner == &token::ID,
       ProgramError::IncorrectProgramId
   );
   ```

CPIデータ注入攻撃を防ぐには、すべての入力データを信頼せず、常に検証することが重要です。特に、ユーザーから提供されるデータや、プログラム間で受け渡されるデータには細心の注意を払いましょう。


# 📝 監査手順書: CPI Data Injection Attack（CPI引数の偽装攻撃）

## 📌 監査目的
- Solanaプログラムがクロスプログラムインボケーション（CPI）を実行する際に渡す引数データが適切に検証されていることを確認する。
- 攻撃者がCPIの引数データを偽装・操作して、意図しない処理を誘発することを防止する。

---

## 🔎 監査観点
監査者は以下の観点でコードを精査すること。

1. **CPI引数の完全性**
   - CPI呼び出し時に渡される引数データが検証され、操作されていないことを確認する。

2. **引数の妥当性検証**
   - 引数データ（数値やバイトデータなど）の範囲や内容が厳格に検証されていることを確認する。

3. **呼び出し先プログラムの検証**
   - CPI呼び出し先プログラムのPubkeyが適切に検証されていることを確認する。

---

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: CPI引数の検証実施確認
- `[ ]` CPIを実行するすべての箇所で、渡される引数データが十分に検証されていることを確認する。
  - 数値の範囲チェック
  - バイトデータの長さや形式チェック

**安全なコード例:**
```rust
require!(amount > 0 && amount <= MAX_AMOUNT, InvalidAmount);
require!(extra_data.len() <= MAX_EXTRA_DATA_LEN, DataTooLarge);
```

### ✔️ Step 2: CPIプログラムIDの検証確認
- `[ ]` CPIを呼び出す前に、呼び出し先プログラムIDが明示的に検証されていることを確認する。

**安全なコード例:**
```rust
require!(
    ctx.accounts.cpi_program.key() == expected_program_id,
    ProgramError::IncorrectProgramId
);
```

### ✔️ Step 3: Anchorによる型安全性の確認
- `[ ]` Anchorフレームワークを使用している場合、型安全なCPI呼び出し（`CpiContext`）を使用していることを確認する。

**安全なコード例（Anchor）:**
```rust
let cpi_ctx = CpiContext::new(
    ctx.accounts.token_program.to_account_info(),
    token::Transfer {
        from: ctx.accounts.source.to_account_info(),
        to: ctx.accounts.destination.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    },
);
token::transfer(cpi_ctx, amount)?;
```

### ✔️ Step 4: 手動CPIを行う場合の理由の明確化（コメント確認）
- `[ ]` Anchorを使わず、手動で`solana_program::program::invoke`を利用してCPIを行っている箇所について、その理由がコメントとして記載されていることを確認する。

**コメントの推奨例:**
```rust
// 手動CPIを使用する理由: Anchor非対応の特殊な処理が必要なため
```

---

## 📂 確認するコード箇所（具体的な確認対象）
- すべてのCPI実行箇所（Anchorによる`CpiContext`および手動の`program::invoke`）
- CPI呼び出しの引数データ検証を行っている関数・コード箇所
- CPI呼び出し先のプログラムID検証箇所

---

## 💬 監査中に確認する質問例
- 「CPIを呼び出すときに、渡すデータの検証は適切に実施されているか？」
- 「呼び出し先のプログラムIDは、期待するプログラムであることが明確に検証されているか？」
- 「Anchorを使わずに手動でCPIを行っている場合、その理由が明確にコメントされているか？」

---

## 🚨 リスク評価基準

**重大な問題として報告すべき場合:**
- CPIの引数データが一切検証されておらず、任意のデータをそのまま渡している場合（攻撃によって不正な操作が実行される可能性）。

**改善推奨として報告すべき場合:**
- CPIプログラムIDの検証が不足している場合（追加検証を推奨）。
- Anchorを利用せず手動CPIを行う際、理由がコードコメントに記載されていない場合（明示的なコメントを推奨）。

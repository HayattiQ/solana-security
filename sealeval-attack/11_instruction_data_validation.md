# Instruction Data Validation（命令データの検証不足）

## 概要

Solanaのプログラムでは、トランザクションに含まれる命令データ（input data）の形式や値が正しいかを検証しないと、攻撃者が不正なデータを注入して想定外の動作を引き起こす可能性があります。

たとえば、無効な数値や型の不整合があってもエラー処理をしないと、予期せぬロジック分岐やメモリ操作につながる恐れがあります。

## 対処法

1. 命令データは必ず正確にデシリアライズし、各フィールドの値の範囲やフォーマットを検証する
2. AnchorやBorshを用いる場合、デシリアライズに失敗した場合は即座にエラーを返す
3. さらに、入力された値が論理的に正しいかどうか（例えば数値の上限・下限など）もチェックする

## 危険なコード例

以下の例では、エラーチェックを十分にせず、直接`try_from_slice`で変換しています：

```rust
// 不正: エラーチェックを十分にせず、直接 try_from_slice で変換している
pub fn insecure_process_instruction(data: &[u8]) -> Result<()> {
    let args: InstructionArgs = InstructionArgs::try_from_slice(data)?; // 失敗しても適切な処理をしていない
    // そのまま args の値を利用して処理
    Ok(())
}
```

## 安全なコード例

以下の例では、デシリアライズのエラーを適切に処理し、さらに値の範囲も検証しています：

```rust
pub fn secure_process_instruction(data: &[u8]) -> Result<()> {
    let args: InstructionArgs = InstructionArgs::try_from_slice(data)
        .map_err(|_| ProgramError::InvalidInstructionData)?;
    // 各フィールドの値の整合性をチェックする
    require!(args.value <= MAX_VALUE, ProgramError::InvalidInstructionData);
    // 正常な場合のみ処理を続行
    Ok(())
}
```

## 追加情報

### 命令データ検証の重要性

命令データの検証は、プログラムのセキュリティにおいて最も基本的かつ重要な要素の一つです。不適切な検証は以下のような問題を引き起こす可能性があります：

1. **バッファオーバーフロー**: デシリアライズ時に適切な境界チェックがないと、メモリ破壊につながる可能性があります
2. **論理的不整合**: 値の範囲チェックがないと、プログラムの内部状態が不正な値になる可能性があります
3. **型の不一致**: 期待する型と異なるデータが渡された場合、予期せぬ動作を引き起こす可能性があります

### 効果的な検証戦略

1. **階層的な検証アプローチ**
   - 構文的検証: データ形式が正しいか（デシリアライズ可能か）
   - 意味的検証: 値が論理的に正しいか（範囲、関係性など）
   - コンテキスト検証: 現在の状態と整合性があるか

2. **Anchorの活用**
   
   Anchorフレームワークは、命令データの検証を簡素化します：
   
   ```rust
   #[derive(Accounts)]
   #[instruction(amount: u64, recipient: Pubkey)]  // 命令データを明示的に定義
   pub struct Transfer<'info> {
       // アカウント定義
   }
   
   pub fn transfer(ctx: Context<Transfer>, amount: u64, recipient: Pubkey) -> Result<()> {
       // amount と recipient は自動的にデシリアライズされ、型チェックされる
       require!(amount > 0 && amount <= MAX_AMOUNT, InvalidAmount);
       // 処理を続行
       Ok(())
   }
   ```

3. **カスタム検証関数**
   
   複雑な検証ロジックは、専用の関数に分離すると可読性が向上します：
   
   ```rust
   fn validate_transfer_params(amount: u64, sender: &Pubkey, recipient: &Pubkey) -> Result<()> {
       require!(amount > 0, InvalidAmount);
       require!(amount <= MAX_AMOUNT, AmountTooLarge);
       require!(sender != recipient, SelfTransferNotAllowed);
       Ok(())
   }
   ```

命令データの厳格な検証は、Solanaプログラムのセキュリティを確保するための基本的な防御策です。すべての入力データに対して「信頼せず、検証する」というアプローチを採用することが重要です。

以下の通り、監査指示書を修正版として書き直しました。  
`#[instruction(...)]`を使っていないこと自体は問題ではなく、必要に応じて推奨する形にしました。

---

# 📝 監査手順書: Instruction Data Validation（命令データの検証不足）

## 📌 監査目的
- Solanaプログラムに渡される命令データ（input data）が適切に検証されているかを確認する。
- 攻撃者が不正な命令データを送ることによるプログラムの予期しない動作を防止する。

## 🔎 監査観点
監査者は以下の観点でコードを精査すること。

1. **デシリアライズの安全性**
   - 命令データが適切な型で厳格にデシリアライズされていることを確認する。

2. **フィールド値の妥当性チェック**
   - デシリアライズ後、各フィールドの値が妥当な範囲内であることを確認する（上限・下限チェック等）。

3. **エラー処理の適切性**
   - 不正なデータや値が渡された場合、適切なエラー処理をしていることを確認する。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: デシリアライズの確認
- `[ ]` 命令データは必ず適切な方法でデシリアライズされていることを確認する（Anchor/Borshなど）。
- `[ ]` デシリアライズ失敗時には即座にエラーが返されるようになっていることを確認する。

**安全な例:**
```rust
let args: InstructionArgs = InstructionArgs::try_from_slice(data)
    .map_err(|_| ProgramError::InvalidInstructionData)?;
```

### ✔️ Step 2: 値の整合性チェックの確認
- `[ ]` デシリアライズされた値が論理的に正しい範囲に収まっていることを確認する（範囲・大小関係・非ゼロチェック等）。

**安全な例:**
```rust
require!(amount > 0 && amount <= MAX_AMOUNT, InvalidAmount);
```

### ✔️ Step 3: Anchorフレームワークを用いている場合の型安全性確認（任意推奨）
- `[ ]` Anchorを使用していて、複雑な命令データを扱う場合、Accounts構造体の`#[instruction(...)]`属性を用いて命令データの型を明示的に指定することを推奨する（必須ではない）。

**推奨する例（Anchor）:**
```rust
#[derive(Accounts)]
#[instruction(amount: u64)]
pub struct Deposit<'info> {
    // アカウント定義
}
```

### ✔️ Step 4: カスタムバリデーション関数の有無と実装の確認
- `[ ]` 複雑な入力検証ロジックが必要な場合、カスタムバリデーション関数が存在し、それが適切に呼び出されていることを確認する。

**安全な例:**
```rust
fn validate_amount(amount: u64) -> Result<()> {
    require!(amount <= MAX_AMOUNT, AmountTooLarge);
    require!(amount > 0, InvalidAmount);
    Ok(())
}
```

## 📂 確認するコード箇所（具体的な確認対象）
- 命令データのデシリアライズ処理箇所
- Anchorを使用している場合で、`#[instruction(...)]`を使っている箇所（使っている場合のみ）
- カスタムバリデーション関数の実装箇所（値の範囲チェック、論理的妥当性チェック）

## 💬 監査中に確認する質問例
- 「命令データのデシリアライズが適切に処理されているか？ 失敗時のエラー処理は十分か？」
- 「渡された値の範囲チェックや不整合のチェックは漏れなく実装されているか？」
- 「複雑なバリデーションがある場合、それが明確かつ分離された関数で実装されているか？」
- 「（Anchor使用時）命令データが複雑な場合に`#[instruction(...)]`属性を使うと型安全性が高まるが、その使用を検討したか？」

## 🚨 リスク評価基準

**重大な問題として報告すべき場合:**
- 命令データのデシリアライズが適切に行われておらず、エラー処理が欠如している場合（メモリ破壊や予期せぬロジック分岐につながる可能性）。

**改善推奨として報告すべき場合:**
- 命令データの範囲チェックが不十分な場合（追加の妥当性検証を推奨）。
- Anchorを使用している場合で、複雑な命令データを扱う際に`#[instruction(...)]`属性を使っていない場合（型安全性向上のため推奨）。  
  ※ 必須ではなく、あくまで推奨事項。
# Rent-Exemption チェックの回避（概要修正版）

## 概要

Solanaでは、各アカウントがネットワーク上に永続的に保存されるために、一定量のSOL（Rent）を保持する必要があります。この最小限のSOLを保持することで、アカウントは「Rent Exempt（レント免除）」状態となり、ネットワーク上で永久に存在し続けられます。

しかし、このRent Exemptチェックを怠ると、アカウントはRent Exempt状態にならず、一定期間経過後（エポック境界）に削除される可能性があります。これにより、攻撃者は意図的にRent不足のアカウントを生成し、そのアカウントが削除されるタイミングを利用した攻撃（復活攻撃や状態の不整合を狙った攻撃）を仕掛ける可能性があります。

特に、Anchorフレームワークを使用している場合、通常はアカウント作成時にRent Exemptが自動的に保証されますが、`rent_exempt = skip`というオプションを明示的に指定すると、Rent不足のアカウント生成が可能になるため、注意が必要です。

## 対処法

1. Anchorを使用する場合、通常は`#[account(init)]`によりRent Exemptが自動的に保証されます。  
   明示的に`rent_exempt = skip`を使う場合は、必ず意図と影響を明確に示したコメントを記載してください。

2. Anchorを使用しない場合は、アカウント作成時やアカウント更新時に、  
   `Rent::get()` を利用してアカウントのRent Exempt状態を必ず確認します。

3. アカウントに保持されるSOLが、アカウントのデータサイズに応じた最低限の量を満たしていることを明示的に検証します。

## 危険なコード例

以下のAnchorコードでは、`rent_exempt = skip`を指定してRent Exempt状態を保証していません：

```rust
// 不正： rent_exempt = skip が指定されており、アカウントがRent不足になる可能性がある
#[derive(Accounts)]
pub struct InsecureCreate<'info> {
    #[account(init, payer = user, space = 8 + Data::LEN, rent_exempt = skip)]
    pub data_account: Account<'info, Data>,
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

このコードはRent不足状態でアカウントを作成可能であり、アカウント削除による復活攻撃などを引き起こすリスクがあります。

## 安全なコード例

通常のAnchorを用いた安全なコード例（デフォルトで安全）：

```rust
// 安全: Anchorのデフォルト挙動によりRent Exemptが保証される
#[derive(Accounts)]
pub struct SecureCreate<'info> {
    #[account(init, payer = user, space = 8 + Data::LEN)]
    pub data_account: Account<'info, Data>,
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

Anchorを使わない場合は以下のように検証します：

```rust
// Anchorを使わない場合の安全なコード例
let rent = Rent::get()?;
require!(
    ctx.accounts.data_account.lamports() >= rent.minimum_balance(Data::LEN),
    ProgramError::InsufficientFunds
);
```

## 追加情報（重要ポイント）

- Anchorを利用する場合、`#[account(init)]`のデフォルト動作でRent Exemptが保証されます。
- 明示的に`rent_exempt = skip`を使う際には、その理由と意図をコメント等で明確に説明しないと、セキュリティ上のリスクを生むため必ず明記してください。
- 通常のケースでは、Anchorの標準機能をそのまま利用することでRent Exempt問題は解消されます。

### セキュリティリスクの具体例

- **アカウント消失**
  - Rent Exemptでないアカウントは一定時間後に削除される可能性があります。
- **復活攻撃（Revival Attack）**
  - 一度削除されたアカウントのアドレスに少額のSOLを送ることで、アカウントが予期せず復活し、プログラム内の状態整合性を破壊します。
- **データの不整合**
  - Rent不足による削除によって、他のアカウントと整合性が取れなくなるリスクがあります。

### 推奨されるベストプラクティス

1. Anchorの標準動作を利用してRent Exemptを常に強制する。
2. 例外的にRent不足を許容するケースでは、その理由を明示するコメントをコード内に残す。
3. 定期的にアカウントのRent状態を確認し、必要ならば追加のSOL送金を行うようロジックを設計する。

Rent Exemptチェックの確実な実施は、Solana上でプログラムの安全性と信頼性を確保するために不可欠です。

# 📝 監査手順書: Rent-Exemption チェックの回避

## 📌 監査目的

アカウント作成時・更新時に、意図せずRent不足のアカウントが生成されないことを保証し、復活攻撃（Revival Attack）やアカウント消失によるデータの不整合を防止する。

## 🔎 監査観点

- Anchorを使用している場合：
  - アカウントの初期化時に、意図せず`rent_exempt = skip`が指定されていないか確認する。
  - 明示的に`rent_exempt = skip`が使用されている場合は、その正当性を確認する。

- Anchorを使用していない場合：
  - アカウント作成・更新時に、明示的にRent Exemptの確認を行っているかを検証する。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Anchorを利用したコードのRent Exempt確認
- 監査対象コードがAnchorを利用している場合、各Accounts構造体の`#[account(init)]`属性を確認する。
- 明示的に`rent_exempt = skip`が指定されている場合は、以下を確認する：
  - なぜRent Exemptをスキップしているのか、その理由がコードコメントに明記されていること
  - skip指定が必要不可欠であることをロジックから判断する
  - 上記を満たさない場合は問題として指摘する

### ✔️ Step 2: Anchorを利用していないコードのRent Exempt確認（該当時のみ）
- Anchorを使用していない場合、アカウント作成・更新処理で明示的にRentのチェック処理が実装されているかを確認する。
  - `Rent::get()`や`minimum_balance()`を使って必要な最低限のlamports量を計算しているかを確認する
  - 上記の検証がない場合は問題として指摘する

## 📂 確認するコード箇所（具体的な確認対象）

- Anchorを使用している場合：
  - `#[account(init, ...)]`を使用しているすべてのAccounts構造体

- Anchorを使用していない場合：
  - `create_account`関数、またはアカウント初期化・更新処理が含まれるすべての箇所

## 💬 監査中に確認する質問例

- 「このアカウントはRent Exemptで作成されていますか？」
- 「`rent_exempt = skip`が意図的に指定されていますが、その理由は何ですか？」
- 「アカウントがRent Exemptでない場合のリスクは理解されていますか？」

## 🚨 リスク評価基準

以下の場合は問題として報告する：

- Anchor使用時：
  - 明確な理由なく`rent_exempt = skip`が指定されている

- Anchor未使用時：
  - Rent Exemptチェックが実装されていない、または不十分

重大度分類：

| 状況 | 重大度 |
|------|--------|
| Rent不足のアカウントが生成される可能性があり、復活攻撃やデータ不整合が起きる | High |
| Rent Exemptではないが、影響範囲が限定的または意図的な場合（理由が明示的） | Medium |
| Anchorで適切なデフォルトが使われている | Low |

## 📝 追加アドバイス（監査者向け）

- Anchor使用のプロジェクトでは、デフォルト挙動によりRent Exemptが保証されているため、`rent_exempt = skip`を明示的に使用している箇所が最大の確認ポイント。
- Anchor未使用の場合は、Rent Exempt状態を維持するための明示的コードが必須である。
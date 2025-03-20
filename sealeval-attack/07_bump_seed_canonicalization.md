# Bump Seed Canonicalization（PDAのBumpシード正規化）

## 概要

Solanaのプログラム派生アドレス(PDA)は種(seed)とバンプ(bump)で生成されます。特定のseedに対し、Solana runtimeは最大値のbumpを使ってPDAを一意に決定しますが、実は複数のbumpが有効となる場合があります。

もしプログラムが利用者から任意のbump値を受け取ってPDAを生成・確認していると、攻撃者は異なる有効なbumpを与えて本来とは別のアドレスを通してしまう可能性があります。つまり、「正規のPDA」ではないアドレスを巧妙にすり替えられる恐れがあります。

## 対処法

常に正規のbumpを使用・検証します。一般的にはプログラム内で`Pubkey::find_program_address`を呼び出して正しいPDAアドレスとbumpを取得し、利用者から渡されたPDAやbumpと照合します。

Anchorでは、PDAを初期化する際に`#[account(seeds = [...], bump)]`と書けば、Anchorテスト時に自動で正規bumpを計算して埋め込んでくれるため、利用者入力に頼る必要がなくなります。どうしてもbumpを引数に取る場合でも、プログラム内で期待するbumpを計算し直して比較することが重要です。

これにより、攻撃者が異なるPDAアドレスを混入させるのを防ぎます。

## 危険なコード例

利用者からseedとbumpを受け取り、自前で`create_program_address`して検証しているケースは危険です。

```rust
pub fn set_value(ctx: Context<SetValue>, seed_key: u64, new_value: u64, user_bump: u8) -> Result<()> {
    let expected_pda = Pubkey::create_program_address(
        &[ seed_key.to_le_bytes().as_ref(), &[user_bump] ],
        ctx.program_id
    )?;
    if expected_pda != ctx.accounts.data_account.key() {
        return Err(ProgramError::InvalidArgument);
    }
    ctx.accounts.data_account.value = new_value;
    Ok(())
}
```

上記では`user_bump`に複数の正解があるケース（例えばあるseedでbump=2とbump=255が共に有効など）があると、攻撃者は別の組み合わせを使って意図しないdata_accountを通過させる恐れがあります。

## 安全なコード例

常にcanonicalなbump（最大値のbump）を使うため、`find_program_address`を利用します。また、渡されたbumpがある場合は照合します。

```rust
pub fn set_value_secure(ctx: Context<SetValue>, seed_key: u64, new_value: u64, user_bump: u8) -> Result<()> {
    let (pda, expected_bump) = Pubkey::find_program_address(
        &[ seed_key.to_le_bytes().as_ref() ],
        ctx.program_id
    );
    require!(pda == ctx.accounts.data_account.key(), InvalidPDA);
    require!(expected_bump == user_bump, InvalidBump);
    ctx.accounts.data_account.value = new_value;
    Ok(())
}
```

上記では`find_program_address`で算出したPDAアドレスとbumpを用い、それと提供された値を比較しています。これにより常に正しいbumpで導出されたPDAのみを受け付け、異なるbumpによる攻撃を防ぎます。

## 追加情報

### PDAとBumpの基本

プログラム派生アドレス（PDA）は、Solanaのプログラムが「所有」するアカウントのアドレスを生成するための仕組みです。PDAは通常のEd25519曲線上にない特殊なアドレスで、以下の要素から生成されます：

1. シード（seeds）：任意のバイト列（複数可）
2. バンプシード（bump）：0～255の値
3. プログラムID：PDAを生成するプログラムのアドレス

### 正規化（Canonicalization）の重要性

Solanaランタイムは、特定のシードとプログラムIDの組み合わせに対して、有効なPDAを生成する最大のバンプ値（通常は255から降順に試行）を使用します。これが「正規の（canonical）」バンプです。

しかし、複数のバンプ値が有効なPDAを生成する場合があります。例えば、あるシードに対してbump=255とbump=254の両方が有効なPDAを生成することがあります。

### 安全なPDA実装のベストプラクティス

1. **常に`find_program_address`を使用する**
   - `create_program_address`ではなく、必ず`find_program_address`を使用して正規のバンプを取得する
   - ユーザー入力のバンプ値を信頼せず、常にプログラム内で再計算する

2. **Anchorの機能を活用する**
   - Anchorの`#[account(seeds = [...], bump)]`構文を使用する

3. **バンプ値の保存と再利用**
   - 初期化時に計算した正規のバンプ値をアカウントに保存する
   - 後続の操作では保存されたバンプ値を使用する
   - 保存されたバンプ値も再検証する

4. **シードの設計に注意する**
   - シードには十分なエントロピー（ランダム性）を持たせる
   - ユーザー固有の情報（Pubkeyなど）をシードに含める
   - シードの長さと構成を慎重に設計する

バンプシードの正規化は、一見すると些細な問題に見えますが、適切に実装しないとセキュリティ上の重大な脆弱性につながる可能性があります。常に正規のバンプを使用し、ユーザー提供の値を信頼しないことが重要です。


# 📝 監査手順書: Bump Seed Canonicalization（PDAのBumpシード正規化）

## 📌 監査目的
- プログラム派生アドレス（PDA）の生成・検証において、正規（canonical）なバンプシードが必ず使用されることを保証する。
- 攻撃者が任意のbump値を用いて、不正なPDAを指定・生成できないことを確認する。

## 🔎 監査観点
監査者は以下の観点でコードを精査する。

### 1. PDA生成時のCanonicalなバンプ値の使用確認
- Anchor非使用時：`Pubkey::find_program_address`を必ず利用しているか確認。
- Anchor使用時：`#[account(seeds = [...], bump)]`が正しく使用されているか確認。

### 2. ユーザー提供のバンプ値の再検証（Anchor非使用時）
- ユーザー入力に依存したバンプ値を直接信頼せず、プログラム内で再計算したバンプ値と比較していることを確認する。

### 3. 保存されたバンプ値の再利用の確認（該当する場合）
- 初期化時に保存したバンプ値を後続の処理で使用する場合、再検証が行われていることを確認する。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: PDA生成ロジックの確認（Anchor非使用の場合のみ）
- [ ] PDA生成コードで、必ず`Pubkey::find_program_address`を使用しているか確認。
- [ ] `Pubkey::create_program_address` を直接使用している場合、明確な理由と必要性が示されているか確認する（原則として`find_program_address`推奨）。
- [ ] ユーザー提供のbump値を直接信用せず、内部で再計算して比較していることを確認。

**安全なコード例:**
```rust
// 明示的にbumpを再検証（意図はセキュリティ向上のため）
let (expected_pda, expected_bump) = Pubkey::find_program_address(&[seeds], ctx.program_id);
require!(expected_pda == ctx.accounts.pda_account.key(), CustomError::InvalidPDA);
require!(expected_bump == user_provided_bump, CustomError::InvalidBump);
```

### ✔️ Step 2: PDA生成ロジックの確認（Anchor使用の場合のみ）
- [ ] Anchorを使用している場合、Accounts構造体内で`#[account(seeds = [...], bump)]`が正しく記述されているか確認する。
- [ ] AnchorのPDA定義以外で、ユーザー提供のbumpを受け取るロジックがある場合、その必要性が明示的に示され、再検証されているか確認（通常はAnchorに任せるべき）。

**安全なAnchorコード例:**
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, seeds = [user.key().as_ref()], bump, payer = user)]
    pub user_data: Account<'info, UserData>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### ✔️ Step 3: 保存したバンプ値の再利用確認（該当する場合）
- [ ] 初期化時にアカウント内に保存したバンプ値を、後続処理で再検証して使用しているか確認する。

**安全な例:**
```rust
// 保存されたbump値の再検証（改ざん防止のため）
require!(stored_bump == recalculated_bump, CustomError::BumpMismatch);
```

### ✔️ Step 4: 明示的なコメントの確認（Anchor非使用など特殊な場合）
- [ ] Anchor非使用時など、特別な理由で明示的にbump値を再検証している場合、コード内にその理由や意図をコメントで明記しているか確認する。
- Anchor使用時（標準的な書き方）はコメント不要です。

## 📂 確認するコード箇所（具体的な確認対象）
- PDAを生成または検証する全ての関数・メソッド
- Anchor使用時のAccounts構造体でのPDA定義
- ユーザー提供のバンプ値を受け取っている箇所
- バンプ値をアカウント内に保存・再利用している箇所

## 💬 監査中に確認する質問例
- 「PDAを生成する際は必ず`find_program_address`を使っていますか？`create_program_address`を使っている場合、その理由は？」
- 「ユーザーが提供したバンプ値をプログラム内部で再計算・再検証していますか？」
- 「アカウント内に保存したバンプ値を後続処理で使用する場合、再検証はしていますか？また、その理由は明記されていますか？」

## 🚨 リスク評価基準
監査中に以下を検出した場合は重大な問題として報告する:

- Anchor非使用時に`create_program_address`を直接使い、ユーザー提供のbump値をそのまま検証せず使用している場合。
- Anchorを使用しているにもかかわらず、`#[account(seeds = [...], bump)]`が未使用でユーザー入力に依存している場合。

軽微だが改善推奨として報告すべき場合:

- Anchor非使用時など特殊なロジックでbump値検証をしているが、その理由や意図がコメントで明示されていない場合（コメント追記を推奨）。

## 🛠 推奨する修正方法

- **Anchor使用時:**
  - `#[account(seeds = [...], bump)]`構文を使用する
  - バンプ値をアカウント内に保存し、後続処理で再利用する

- **Anchor非使用時:**
  - `create_program_address`ではなく`find_program_address`を使用する
  - ユーザー提供のバンプ値を再検証する
  - バンプ値の検証ロジックにコメントを追加する

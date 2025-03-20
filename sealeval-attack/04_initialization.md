# Initialization（初期化に関する攻撃: フロントラン & 再初期化）

## 概要

アカウントの初期化処理にも注意が必要です。初期化に絡む代表的な攻撃には「フロントラン攻撃による不正初期化」と「再初期化攻撃」があります。

### 初期化フロントラン攻撃

プログラム展開直後など、最初の初期化トランザクションを他者に先を越される（フロントラン）恐れがあります。例えばグローバルな設定アカウントをinitしようとしている場合、本来はデプロイ者が最初に呼ぶべきinitializeを、悪意ある第三者が先に実行して初期化してしまうと、以降正規ユーザはそのアカウントを利用できず機能が乗っ取られる形になります。実際、initializerの権限チェックがないと誰でも初期化できてしまうため、悪用される可能性があります。

### 再初期化攻撃

Anchorの`init_if_needed`などを不用意に使うと、既に存在するアカウントを再度初期化し直すことができてしまいます。攻撃者が本来一度きりの初期化関数を何度も呼び出せると、データを上書きしたり意図しない振る舞いを引き起こせます。例えば一度初期化されたはずのアカウントに対しもう一度`init_if_needed`で処理が走ると、元のデータが破壊される恐れがあります。

## 対処法

初期化処理には以下の対策を講じます。

### 権限のある初期化者のみ実行可能にする

グローバル初期化や重要アカウントの初期化には、特定の署名者だけが呼べるように制限します。Anchorでは`#[account(constraint = signer.key() == expected_pubkey)]`などでチェックできます。例えばプログラムのアップグレード権限者(PDAのアップグレード権限)のみが初期化できるようにする実装が有効です。これにより第三者による初期化フロントランを防ぎます。

### 二重初期化の防止

一度初期化したアカウントに対して再度初期化処理が走らないようにします。具体的には、Anchorの`init_if_needed`の使用は可能な限り避け、どうしても使う場合はアカウント構造体に`is_initialized`フラグ（bool値）を持たせて既に初期化済みかどうか記録し、再初期化要求時にはエラーを出すようにします。Anchor v0.26以降では`init_if_needed`使用時にもいくつか自動チェックが入りますが、安全のため自前でフラグ管理する方法が推奨されます。

## 危険なコード例

初期化フロントランの脆弱な例として、誰でも呼べるinitialize関数でグローバル設定アカウントを生成しているケースを示します（権限者チェックなし）:

```rust
// 不正: 誰でもglobal_configを初期化できてしまう
#[derive(Accounts)]
pub struct InitializeConfig<'info> {
    #[account(init, payer = signer, space = 8 + Config::LEN, seeds = [b"config"], bump)]
    pub global_config: Account<'info, Config>,
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

pub fn initialize_config(ctx: Context<InitializeConfig>) -> Result<()> {
    ctx.accounts.global_config.value = 0;
    Ok(())
}
```

上記ではsignerはチェックされていますが、signerが誰でも良いため、先に呼んだ者勝ちになります。

また、再初期化の脆弱な例として、既存アカウントを`init_if_needed`で上書きしてしまうケースです:

```rust
// 不正: init_if_neededを使い、二度目の呼び出しで既存データが上書きされうる
#[derive(Accounts)]
pub struct InitializeUser<'info> {
    #[account(mut, init_if_needed, payer = creator, space = 8 + User::LEN, seeds=[b"user", creator.key().as_ref()], bump)]
    pub user_account: Account<'info, User>,
    pub creator: Signer<'info>,
    pub system_program: Program<'info, System>,
}

pub fn initialize_user(ctx: Context<InitializeUser>, data: UserData) -> Result<()> {
    let user = &mut ctx.accounts.user_account;
    user.data = data;  // 2回目以降の呼び出しでは既存ユーザーデータが上書きされてしまう
    Ok(())
}
```

## 安全なコード例

初期化フロントラン対策として、例えば以下のように特定のPubkeyのみが初期化可能とします。

```rust
#[derive(Accounts)]
pub struct InitializeConfigSecure<'info> {
    #[account(
        init,
        payer = signer,
        space = 8 + Config::LEN,
        seeds = [b"config"], bump
    )]
    pub global_config: Account<'info, Config>,
    #[account(constraint = signer.key() == allowed_authority.key)]  // 特定の権限者のみ許可
    pub signer: Signer<'info>,
    // allowed_authority はあらかじめprogram_dataから取得したアップグレード権限のPubkeyなど
    // ...（省略）...
}
```

こうすることでsignerが指定のPubkeyでなければ初期化トランザクション自体が失敗します。

再初期化対策としては、Anchorではなくプログラム内でフラグをチェックする手段の一例を示します。

```rust
pub fn initialize_user_secure(ctx: Context<InitializeUserSecure>, data: UserData) -> Result<()> {
    let user = &mut ctx.accounts.user_account;
    require!(!user.is_initialized, AlreadyInitialized);  // すでに初期化済みならエラー
    user.data = data;
    user.is_initialized = true;
    Ok(())
}
```

このように状態フラグ`is_initialized`を用いることで、2回目以降の初期化処理を明確に拒否できます。

## 追加情報

初期化処理のセキュリティを強化するための追加ポイント：

### フロントラン対策の詳細

1. 初期化権限の厳格な管理
   - プログラムのアップグレード権限者のみに初期化を許可
   - マルチシグを使用した初期化権限の分散
   - 特定のPDAのみが初期化できるよう設計

2. 初期化トランザクションの保護
   - 可能な限り非公開環境で初期化を完了
   - 初期化トランザクションを優先的に処理してもらうための工夫（優先手数料など）

### 再初期化対策の詳細

1. 初期化状態の明示的な管理
   - アカウント構造体に`is_initialized`フラグを追加
   - 初期化時に特殊な値や署名を記録
   - 初期化時刻を記録し、一定期間後の再初期化のみ許可（必要な場合）

2. Anchorの安全な使用
   - `init_if_needed`の代わりに`init`と条件分岐を使用
   - Anchor v0.26以降の改善された安全機能を活用
   - カスタムバリデーションを追加

初期化処理は、プログラムの状態管理の基盤となる重要な部分です。適切な権限チェックと再初期化防止策を組み合わせることで、初期化に関連する攻撃から保護することができます。


# 📝 監査手順書: Initialization（初期化に関する攻撃: フロントラン & 再初期化）

---

## 📌 監査目的

- アカウントの初期化処理に対するフロントラン攻撃や再初期化攻撃を防止するため、適切なチェックが実装されていることを保証する。
- 特定の権限者のみが初期化を実施でき、一度初期化されたアカウントが再度初期化されないよう対策されていることを確認する。
- 特に`init_if_needed`を使っている箇所について、その使用が本当に必要かを評価し、原則として`init`の使用を推奨する。

---

## 🔎 監査観点

監査者は以下の観点でコードを精査する。

### 1. 初期化権限の検証

- 初期化処理を呼べる署名者が特定のPubkeyまたは特定の権限者のみであることを確認する。

### 2. フロントラン攻撃への対策

- グローバル設定や重要な初期化処理を第三者に先取りされないようになっていることを確認する。

### 3. 再初期化防止措置の確認

- 一度初期化されたアカウントが再度初期化されないように対策が講じられていることを確認する。
- Anchorの`init_if_needed`使用時に状態フラグを導入するなど、明示的な再初期化防止措置があることを確認する。
- `init_if_needed`を使用する必要性が本当にあるかを評価する（可能な限り`init`の利用を推奨）。

---

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: 初期化処理の署名者・権限確認

- [ ] グローバルな初期化処理を行う関数に、特定の署名者のみが許可されていることを確認する（例：アップグレード権限者や特定のPDAのみ初期化許可）。
- [ ] Anchor使用の場合、アカウント制約（`constraint`）や`has_one`などを用いて署名者を適切に検証していることを確認する。

**良い例:**

```rust
#[derive(Accounts)]
pub struct InitializeConfig<'info> {
    #[account(init, payer = signer, seeds = [b"config"], bump)]
    pub global_config: Account<'info, Config>,
    #[account(constraint = signer.key() == allowed_authority)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### ✔️ Step 2: 再初期化防止措置の有無

- [ ] Anchorの`init_if_needed`を使用している場合、再初期化攻撃を防ぐためのフラグ（例：`is_initialized`）がアカウント構造体内に定義されているかを確認する。
- [ ] 初期化関数内でフラグを検証し、すでに初期化済みの場合は処理を拒否することを確認する。

**良い例:**

```rust
pub fn initialize_user(ctx: Context<InitializeUser>, data: UserData) -> Result<()> {
    let user = &mut ctx.accounts.user_account;
    require!(!user.is_initialized, ErrorCode::AlreadyInitialized);
    user.data = data;
    user.is_initialized = true;
    Ok(())
}
```

### ✔️ Step 3: `init_if_needed`の使用妥当性評価

- [ ] コード内でAnchorの`init_if_needed`を使っている箇所をすべて特定する。
- [ ] 各箇所について、`init_if_needed`を本当に使う必要があるのか評価し、明確な理由がない限り`init`に置き換えるよう推奨する。
- [ ] もし`init_if_needed`が必要な場合、その理由と再初期化防止策がコード内またはドキュメントに明示されていることを確認する。

**推奨する監査者コメント例:**
> 「ここでの`init_if_needed`の使用理由が不明確です。特段の理由がなければ、より安全な`init`を使用することを強く推奨します。」

---

## 📂 確認するコード箇所（具体的な確認対象）

- `#[account(init)]` または `#[account(init_if_needed)]` を使っているすべての構造体定義箇所
- 初期化関数の冒頭部分（署名者の制約チェックの有無）
- グローバル設定アカウントや重要アカウントを初期化する処理
- 状態フラグ（`is_initialized`）を参照するコード箇所
- `init_if_needed`を利用しているコードの妥当性評価対象箇所

---

## 💬 監査中に確認する質問例

- 「この初期化関数は特定の署名者以外から呼び出すことは絶対に不可能ですか？」
- 「グローバル設定アカウントをフロントラン攻撃から守るため、権限チェックは十分ですか？」
- 「再初期化を防ぐためにどのような仕組みが用意されていますか？」
- 「Anchorの`init_if_needed`を使用している理由は何ですか？これは必ず必要ですか？initを使えない明確な理由はありますか？」

---

## 🚨 リスク評価基準

監査中に以下を検出した場合は重大な問題として報告する:

- グローバルアカウントや重要な初期化処理が誰でも呼び出せてしまう状態
- Anchorの`init_if_needed`を使用しているが、再初期化防止策が存在しない場合
- 初期化時の権限チェックが欠落しており、フロントラン攻撃を許容する可能性がある場合
- `init_if_needed`が特に理由なく使用されており、`init`への置き換えが可能である場合

---

## 🛠 推奨する修正方法

- **フロントラン攻撃対策:**
  - 初期化関数に特定の署名者のみが呼び出せるよう制約を追加する
  - グローバル設定の初期化には、プログラムのアップグレード権限者のみが実行できるよう制限する

- **再初期化攻撃対策:**
  - 可能な限り`init_if_needed`の使用を避け、`init`を使用する
  - アカウント構造体に`is_initialized`フラグを追加し、初期化関数内でチェックする
  - 初期化済みのアカウントに対する再初期化要求はエラーを返すようにする

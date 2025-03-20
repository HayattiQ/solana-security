# PDA Sharing（PDAの不適切な共有）

## 概要

一つのPDAを複数のユーザやコンテキストで使い回す設計はセキュリティ上危険です。PDA（プログラム派生アドレス）は任意のseedで生成できますが、不用意に共通のseedを使っていると、本来分離すべき権限ドメインが混ざり合ってしまいます。

例えば本来ユーザごとに異なるPDAを用いるべき場面で、全ユーザ共通のPDAをAuthorityにしていた場合、あるユーザが他のユーザの資金やデータにアクセスできてしまう可能性があります。また、AnchorのSeedsを使った署名（signer_seedsによるCPI署名）でも、PDAの種に不適切な値を使うと他人のPDAで署名できてしまうケースがあります。

## 対処法

PDAは用途ごと・ユーザごとにユニークになるよう設計し、さらにPDAに関連付けられた権限者フィールドを検証します。各Liquidityプールや各ユーザごとに異なるseed組み合わせを用いてPDAを生成するのが理想です。

また、PDAの内部データに「このPDAの権限者」が記録されている場合、それが処理を呼び出しているユーザと一致するかを`has_one`でチェックするようにします。Anchorでは`#[account(has_one = user)]`のようにするだけで、PDA内のフィールドが正しいユーザPubkeyと一致していることを保証できます。

要は、PDAごとの権限範囲を明確に区切り、検証することで、不正な共有を防ぎます。

## 危険なコード例

攻撃例として、ある「metadata_account」というPDAを使ってトークンの移転を承認する仕組みがあったとします。metadata_accountには`creator`フィールドがあり本来その人だけが操作できる想定ですが、コードでそれをチェックしていない場合、攻撃者は他人のmetadata_accountを使って自分に有利な操作を行えます：

```rust
// 不正: metadata_accountのcreatorと署名者の照合がない
pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>, amount: u64) -> Result<()> {
    let seeds = &[&[b"metadata_account", ctx.accounts.metadata_account.creator.as_ref(), &[ctx.bumps.metadata_account]]];
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.vault.to_account_info(),
            to: ctx.accounts.dest.to_account_info(),
            authority: ctx.accounts.metadata_account.to_account_info(),
        },
        seeds,
    );
    token::transfer(cpi_ctx, amount)?;  // PDA(metadata_account)が署名権限を持つ
    Ok(())
}
```

上記では、攻撃者は別のユーザが所有するmetadata_accountを引数に渡し、自分のトランザクションで署名（signerは自分）しつつ、そのPDAを署名者シードとして利用することで、本来他人のVaultから資金を移動させる、といった攻撃が可能です（コード上チェックがないため）。

## 安全なコード例

metadata_accountに`creator`フィールドがあるなら、Anchorの`has_one`で`creator`と署名者を結びつけます。また、Withdraw処理呼び出し時にも明示的にチェックすると安心です。

```rust
#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    pub creator: Signer<'info>,
    #[account(seeds = [b"metadata_account", metadata_account.creator.as_ref()], bump, has_one = creator)]
    pub metadata_account: Account<'info, MetadataAccount>,
    // 他のアカウント（vaultやdestなど）は省略
}

pub fn secure_withdraw(ctx: Context<SecureWithdraw>, amount: u64) -> Result<()> {
    // ここではhas_oneによりmetadata_account.creator == creatorが保証されている
    // あとは同様にCPIでtransferする
    // ...
    Ok(())
}
```

このように、PDA内に保持している権限者（ここでは`metadata_account.creator`）と実際の署名者を一致させるチェックを入れることで、攻撃者が他人のPDAを悪用することを防ぎます。また、PDAのseed設計段階からユーザ固有情報（ユーザのPubkeyなど）を組み込んでおけば、本質的に他人のPDAを使い回すことはできなくなります。

## 追加情報

### PDA設計のベストプラクティス

1. **ユーザー固有のシードを使用する**

   PDAを生成する際は、ユーザーのPubkeyや一意の識別子をシードに含めることで、各ユーザーに固有のPDAを作成します。

   ```rust
   // 良い例: ユーザー固有のPDA
   #[account(
       seeds = [b"user_data", user.key().as_ref()],
       bump
   )]
   pub user_data: Account<'info, UserData>,
   ```

2. **コンテキスト固有のシードを使用する**

   異なる機能や目的に対して、異なるシードプレフィックスを使用します。

   ```rust
   // 良い例: 機能ごとに異なるPDA
   #[account(
       seeds = [b"user_profile", user.key().as_ref()],
       bump
   )]
   pub profile: Account<'info, UserProfile>,

   #[account(
       seeds = [b"user_settings", user.key().as_ref()],
       bump
   )]
   pub settings: Account<'info, UserSettings>,
   ```

3. **PDA内の権限フィールドを検証する**

   PDAに権限者フィールドを持たせ、操作時に必ずチェックします。

   ```rust
   #[account]
   pub struct UserData {
       pub owner: Pubkey,  // 権限者フィールド
       // その他のデータ
   }

   #[derive(Accounts)]
   pub struct UpdateUserData<'info> {
       pub user: Signer<'info>,
       #[account(
           mut,
           seeds = [b"user_data", user.key().as_ref()],
           bump,
           constraint = user_data.owner == user.key()  // 権限者チェック
       )]
       pub user_data: Account<'info, UserData>,
   }
   ```

4. **PDAの権限範囲を明確に定義する**

   各PDAが持つ権限と責任の範囲を明確に定義し、必要以上の権限を持たせないようにします。

### 共通の落とし穴

1. **グローバルPDAの過剰な使用**

   プログラム全体で単一のグローバルPDAを使用すると、権限の分離が難しくなります。代わりに、必要に応じて複数の特定目的PDAを使用しましょう。

2. **シードの不十分なエントロピー**

   シードに十分なエントロピー（ランダム性や一意性）がないと、PDAの衝突や予測可能性の問題が発生する可能性があります。

3. **PDA間の権限漏洩**

   あるPDAの権限を別のPDAに漏らさないよう、各PDAの責任範囲を明確に分離します。

PDAの適切な設計と使用は、Solanaプログラムのセキュリティにおいて非常に重要です。各PDAの権限範囲を明確に定義し、ユーザーごとに固有のPDAを使用することで、多くのセキュリティリスクを軽減できます。


# 📝 監査手順書: PDA Sharing（PDAの不適切な共有）

## 📌 監査目的
- プログラム派生アドレス（PDA）がユーザー間や機能間で不適切に共有されていないことを確認する。
- PDAの権限が正しくユーザーや用途ごとに分離されていることを保証する。

## 🔎 監査観点
監査者は以下の観点でコードを精査する。

### 1. PDAのユニーク性の確認
- PDA生成のためのシードがユーザーごと、または用途ごとに一意であることを確認する。

### 2. PDAにおける権限者チェック
- PDAの内部データに、操作を許可される権限者（ownerやauthority）が明示されており、適切に検証されていることを確認する。

### 3. PDA署名 (signer_seeds) の正当性
- CPI呼び出し等でPDAを署名者として使用する場合、署名するPDAが必ず正しい権限範囲のものであるかを確認する。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: PDA生成時のシード設計確認
- [ ] PDAの生成に使用されているシードがユーザー固有または用途固有であり、異なるユーザーや用途間で共有されていないことを確認する。

**安全なシード設計例:**
```rust
#[account(
    seeds = [b"user_data", user.key().as_ref()], 
    bump
)]
pub user_data: Account<'info, UserData>,
```

**不適切なシード設計例（共通化しすぎ）:**
```rust
#[account(
    seeds = [b"global_vault"], 
    bump
)]
pub vault: Account<'info, Vault>,
```

### ✔️ Step 2: PDA内の権限者フィールドの確認
- [ ] PDA内に権限者（authority, owner, creator等）のフィールドが存在し、それがAccounts構造体の`has_one`または`constraint`などで適切に検証されていることを確認する。

**安全な例:**
```rust
#[derive(Accounts)]
pub struct UpdateAccount<'info> {
    #[account(
        mut,
        seeds = [b"user_account", authority.key().as_ref()],
        bump,
        has_one = authority
    )]
    pub user_account: Account<'info, UserAccount>,
    pub authority: Signer<'info>,
}
```

**不適切な例（権限者検証なし）:**
```rust
#[account(seeds = [b"user_account"], bump)]
pub user_account: Account<'info, UserAccount>,
```

### ✔️ Step 3: CPI署名時のPDAシード確認
- [ ] PDAを署名者としてCPI (invoke_signed) を行う際に、使用する`signer_seeds`が適切なユーザーやコンテキスト固有のものであり、不適切に共有されていないことを確認する。

**安全なCPI署名例:**
```rust
let seeds = &[
    b"user_vault",
    user.key().as_ref(),
    &[vault_bump]
];
invoke_signed(&ix, accounts, &[&seeds[..]])?;
```

**不適切なCPI署名例（共通のPDAを使用）:**
```rust
let seeds = &[
    b"global_vault", // 全ユーザー共通のseed
    &[vault_bump]
];
invoke_signed(&ix, accounts, &[&seeds[..]])?;
```

## 📂 確認するコード箇所（具体的な確認対象）
- PDAを生成する全てのコード（AnchorのAccounts構造体の`#[account(seeds = [...])]`を含む）
- PDAを署名者として利用しているCPI呼び出し箇所
- PDAアカウントのデータ構造における権限フィールド（owner、authorityなど）が存在する全ての箇所

## 💬 監査中に確認する質問例
- 「PDA生成に用いられるシードは、ユーザーごとに固有のものでしょうか？　用途間で共通化されていませんか？」
- 「PDA内に権限者が記録されている場合、それを実際に検証していますか？」
- 「CPI署名時に使用されるPDAは、特定のユーザーやコンテキストに限定されていますか？ 共通化されたPDAが不適切に使われる恐れはありませんか？」

## 🚨 リスク評価基準
監査中に以下を検出した場合は重大な問題として報告する:

- 全ユーザー共通またはコンテキスト間で共有されているPDAが、資産や権限操作を行う署名に用いられている場合（資産損失、データ改ざん等の可能性がある）。

軽微だが改善推奨として報告すべき場合:

- PDA内に権限者フィールドがあるが、`has_one`や`constraint`属性での検証が明示されていない場合（検証を推奨）。
- シード設計がユーザー固有でない場合（将来的なリスク軽減のため改善を推奨）。

## 🛠 推奨する修正方法

- **PDAのシード設計改善:**
  - ユーザー固有の情報（Pubkey等）をシードに含める
  - 機能や用途ごとに異なるシードプレフィックスを使用する

- **権限者検証の追加:**
  - PDA内に権限者フィールドを追加する
  - Accounts構造体で`has_one`や`constraint`を使用して検証する

- **CPI署名の安全性向上:**
  - ユーザー固有のPDAのみを署名に使用する
  - 共通PDAの使用を避け、権限を適切に分離する

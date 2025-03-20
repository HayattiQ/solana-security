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
   - Anchorの`seeds_with_nonce`関数を使用してPDAを検証する

3. **バンプ値の保存と再利用**
   - 初期化時に計算した正規のバンプ値をアカウントに保存する
   - 後続の操作では保存されたバンプ値を使用する
   - 保存されたバンプ値も再検証する

4. **シードの設計に注意する**
   - シードには十分なエントロピー（ランダム性）を持たせる
   - ユーザー固有の情報（Pubkeyなど）をシードに含める
   - シードの長さと構成を慎重に設計する

バンプシードの正規化は、一見すると些細な問題に見えますが、適切に実装しないとセキュリティ上の重大な脆弱性につながる可能性があります。常に正規のバンプを使用し、ユーザー提供の値を信頼しないことが重要です。

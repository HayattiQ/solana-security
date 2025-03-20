# Closing Accounts（アカウント閉鎖時の不備と復活攻撃）

## 概要

不要になったアカウントを閉じる際にも注意が必要です。単純にラムポートを移動しデータをゼロクリアしただけではアカウントを完全に消去したことになりません。Solanaではアカウントの実態（アドレス）は残り得るため、攻撃者がそこに再度少額のラムポートを送り込むことで「復活」させることが可能です。

いわゆる**Revival Attack（復活攻撃）**で、閉じたはずのアカウントが蘇り、プログラム内の他の箇所で誤って参照されたり、再利用されてしまう恐れがあります。例えば一度閉じたアカウントに対し、残っていたデータを悪用して再度取引が行われる、といったシナリオが考えられます。

## 対処法

Anchorの`close`属性を使ってアカウントを閉じるのが簡潔かつ安全です。`close`を使用すると以下を自動で行います:

1. アカウントの全ラムポートを指定した受取先に移動する（残高を0にする）。
2. アカウントのデータをすべて0にクリアする。
3. アカウントのディスクリミネータを特殊な「CLOSED」状態に設定する。

特に3の処理によって、閉じられたアカウントに外部からラムポートを送り込んで再利用しようとしても、ディスクリミネータをチェックする箇所で無効と判断できるようになります。

Anchorを使わない場合は、自前で上記と同等の処理を実装し、他の関数でそのアカウントが閉じられていないか（ディスクリミネータを確認する等）をチェックする必要があります。

## 危険なコード例

ラムポートを移してデータを消すだけの閉鎖処理は不十分です。

```rust
// 不正: アカウントを閉じる際のディスクリミネータ設定がない
pub fn close_account(ctx: Context<CloseAccount>) -> Result<()> {
    let target = &mut ctx.accounts.target;
    let dest = &mut ctx.accounts.destination;
    **dest.lamports.borrow_mut() += **target.lamports.borrow();  // 残高移転
    **target.lamports.borrow_mut() = 0;                          // ラムポート0
    target.data.fill(0);  // データゼロクリア (疑似コード)
    Ok(())
}
```

一見問題なさそうですが、閉じたtargetアカウントに対し攻撃者が後でわずかなSOLを送ると、targetは残高を持ちデータ長も維持したまま「存在」できます。プログラム内でこのtargetを再度引数にとると、もしディスクリミネータチェックをしていなければ古いデータ（ゼロクリアしたとはいえ何かしらの構造体として解釈されうる）を読み込んでしまう可能性があります。

## 安全なコード例

Anchorでは以下のように`close`を指定します。

```rust
#[derive(Accounts)]
pub struct CloseAccount<'info> {
    #[account(mut, close = destination)]
    pub target: Account<'info, Data>,       // 閉じるアカウント
    pub destination: Signer<'info>,         // 残高を受け取る先（ここでは単純に署名者）
}
```

こうするだけで、処理終了時にAnchorがtargetのlamportsをdestinationにすべて移動し、targetのデータをゼロクリアし、かつディスクリミネータをCLOSED_ACCOUNT_DISCRIMINATORに書き換えてくれます。結果、targetアカウントは完全に閉鎖状態となり、仮に復活させようとしてもプログラム側で弾けるようになります。

Anchorを使わない場合も、閉鎖処理では必ずディスクリミネータを書き換えることが推奨されます。具体的には、アカウントデータの先頭8バイトにAnchorのCLOSED_ACCOUNT_DISCRIMINATOR（[0xFF; 8]）を書き込む実装を自前で行います。さらに、他の関数でもそのディスクリミネータをチェックし、閉鎖済みアカウントが渡された場合はエラーにするなどの対策が有効です。

## 追加情報

### アカウント閉鎖の完全な理解

Solanaでは、アカウントを「閉じる」という操作は実際にはブロックチェーン上からアカウントを完全に削除するわけではありません。代わりに、以下の操作の組み合わせによってアカウントを「閉じた」状態にします：

1. **ラムポート（SOL）の移動**：アカウントの存続には最小限のSOLが必要なため、すべてのSOLを別のアカウントに移動させることで、アカウントは「レント免除」の状態を失います。
2. **データのクリア**：アカウントのデータをゼロで埋めることで、以前の状態情報を消去します。
3. **ディスクリミネータの変更**：アカウントが「閉じられた」ことを示す特殊な値をディスクリミネータ（通常はデータの最初の8バイト）に設定します。

### 復活攻撃（Revival Attack）の詳細

復活攻撃は以下のステップで行われます：

1. 攻撃者は閉じられたアカウントのアドレスを特定します。
2. そのアドレスに少量のSOLを送金し、アカウントを「復活」させます。
3. プログラムが閉じられたアカウントのチェックを適切に行っていない場合、攻撃者は復活したアカウントを使用して不正な操作を行います。

### 安全なアカウント閉鎖のベストプラクティス

1. **Anchorの`close`属性を使用する**
   - Anchorの`close`属性は、ラムポートの移動、データのクリア、ディスクリミネータの設定を自動的に行います。
   - 可能な限りこの機能を活用しましょう。

2. **手動で閉じる場合の完全な実装**
   ```rust
   pub fn manual_close(ctx: Context<ManualClose>) -> Result<()> {
       let target = &mut ctx.accounts.target;
       let dest = &mut ctx.accounts.destination;
       
       // 1. ラムポートの移動
       let target_lamports = **target.lamports.borrow();
       **target.lamports.borrow_mut() = 0;
       **dest.lamports.borrow_mut() += target_lamports;
       
       // 2. データのクリア
       let mut data = target.try_borrow_mut_data()?;
       for byte in data.iter_mut() {
           *byte = 0;
       }
       
       // 3. ディスクリミネータの設定（最初の8バイト）
       if data.len() >= 8 {
           data[0..8].copy_from_slice(&[0xFF; 8]); // CLOSED_ACCOUNT_DISCRIMINATOR
       }
       
       Ok(())
   }
   ```

3. **アカウント使用前の検証**
   - アカウントを使用する前に、そのアカウントが閉じられていないことを確認します。
   - Anchorを使用する場合、`Account<'info, T>`型は自動的にディスクリミネータをチェックします。
   - 手動で検証する場合は、以下のようなコードを使用します：
   
   ```rust
   pub fn check_not_closed(account: &AccountInfo) -> Result<()> {
       let data = account.try_borrow_data()?;
       if data.len() >= 8 {
           let discriminator = &data[0..8];
           if discriminator == &[0xFF; 8] {
               return Err(ProgramError::AccountClosedAlready.into());
           }
       }
       Ok(())
   }
   ```

アカウントの適切な閉鎖と検証は、Solanaプログラムのセキュリティにおいて重要な要素です。特にAnchorフレームワークを使用する場合は、提供される安全な閉鎖メカニズムを活用することで、多くのリスクを軽減できます。


# 📝 監査手順書: Closing Accounts（アカウント閉鎖時の不備と復活攻撃）

## 📌 監査目的
- 不要になったアカウントが安全かつ完全に閉鎖されていることを確認する。
- 復活攻撃（Revival Attack）が発生しないことを保証する。

## 🔎 監査観点
監査者は以下の観点でコードを精査すること。

1. **アカウント閉鎖処理の妥当性**
   - Anchorの`close`属性を使用しているか、または手動の場合は適切なディスクリミネータ処理が含まれているかを確認する。

2. **アカウント閉鎖後の再利用防止**
   - 閉鎖済みのアカウントを誤って再度使用しないよう、アカウント使用前にディスクリミネータの検証が行われていることを確認する。

3. **ラムポートの完全移動とデータのクリア**
   - アカウント閉鎖時にラムポートが完全に移動され、データがゼロクリアされていることを確認する。

## 📋 監査手順詳細（チェックリスト）

### ✔️ Step 1: Anchor使用時のアカウント閉鎖確認
- `[ ]` Anchorフレームワークを使用している場合、アカウント閉鎖には`close`属性が必ず使用されていることを確認する。

**安全な例（推奨）：**
```rust
#[derive(Accounts)]
pub struct CloseAccount<'info> {
    #[account(mut, close = destination)]
    pub target: Account<'info, Data>,
    pub destination: Signer<'info>,
}
```

### ✔️ Step 2: Anchor非使用時のアカウント閉鎖確認
- `[ ]` Anchorを使わずに手動で閉鎖する場合、以下の処理がすべて行われていることを確認する。
    - `[ ]` ラムポートを全額別アカウントに移動
    - `[ ]` アカウントデータの完全ゼロクリア
    - `[ ]` ディスクリミネータを閉鎖済み値（[0xFF;8]）に設定

**安全な例（手動で閉じる場合）：**
```rust
pub fn manual_close(ctx: Context<ManualClose>) -> Result<()> {
    let target = &mut ctx.accounts.target;
    let dest = &mut ctx.accounts.destination;

    let target_lamports = **target.lamports.borrow();
    **target.lamports.borrow_mut() = 0;
    **dest.lamports.borrow_mut() += target_lamports;

    let mut data = target.try_borrow_mut_data()?;
    for byte in data.iter_mut() {
        *byte = 0;
    }
    if data.len() >= 8 {
        data[0..8].copy_from_slice(&[0xFF; 8]); 
    }

    Ok(())
}
```

### ✔️ Step 3: アカウント再利用防止のチェック
- `[ ]` 閉鎖したアカウントを他の処理で再利用する前に、必ずディスクリミネータをチェックしていることを確認する（Anchorの場合は自動で行われる）。
- `[ ]` Anchor非使用時は手動でディスクリミネータ確認処理を実装していることを確認する。

**安全な例（手動で確認する場合）：**
```rust
pub fn check_not_closed(account: &AccountInfo) -> Result<()> {
    let data = account.try_borrow_data()?;
    if data.len() >= 8 && &data[0..8] == &[0xFF; 8] {
        return Err(ProgramError::AccountClosedAlready.into());
    }
    Ok(())
}
```

## 📂 確認するコード箇所（具体的な確認対象）
- Anchorの`#[account(close = ...)]`が使われている箇所。
- 手動でアカウント閉鎖を行っている関数の実装。
- 閉鎖済みアカウントを再利用する可能性がある全ての箇所（再利用防止チェック）。

## 💬 監査中に確認する質問例
- 「アカウントを閉じる際にAnchorの`close`属性を使用していますか？使用しない場合、ラムポート移動、データクリア、ディスクリミネータ設定の処理が行われていますか？」
- 「閉じたアカウントが再利用される危険性はありませんか？再利用防止のチェックは明確に行われていますか？」

## 🚨 リスク評価基準

**重大な問題として報告すべき場合:**
- アカウント閉鎖時にAnchorの`close`属性がなく、かつディスクリミネータの設定がされていない場合（復活攻撃が可能となる）。

**改善推奨として報告すべき場合:**
- 閉鎖したアカウントを再度使用する際に、明示的なディスクリミネータチェックが不足している場合（追加のチェックを推奨）。


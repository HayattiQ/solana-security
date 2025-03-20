# 🔍 攻撃ベクトル監査レポート

## ✅ 1. 攻撃ベクトル名とID

- **攻撃ベクトル:** `#0 - Signer Authorization（署名者チェックの欠如）`

---

## ✅ 2. 概要（Description）

トランザクションの署名者であること自体に任せきり、プログラム内で適切な署名者の検証を行わないと、想定外のアカウントが権限を持つ操作を実行できてしまいます。つまり「権限を持つはずのアカウント」が実際に署名しているか確認しないと、誰でもその権限を偽装して操作を実行できる恐れがあります。

特にステーキングプログラムでは、管理者権限を持つ操作や、ユーザー固有のステーキングデータへのアクセスにおいて、適切な署名者チェックが不可欠です。

---

## ✅ 3. 監査対象コード箇所（Code Locations）

| No. | ファイル名     | 行番号     | 関数名・Accounts構造体名 |
|-----|----------------|------------|-------------------------|
| 1   | `instructions/stake_token.rs` | 15-67行目 | `StakeTokenInstruction` |
| 2   | `instructions/stake_token.rs` | 69-152行目 | `handle_stake_token` |
| 3   | `instructions/unstake_token.rs` | 15-67行目 | `UnstakeTokenInstruction` |
| 4   | `instructions/unstake_token.rs` | 69-152行目 | `handle_unstake_token` |
| 5   | `instructions/admin_fn.rs` | 15-32行目 | `SetAdmin` |
| 6   | `instructions/admin_fn.rs` | 34-51行目 | `SetStakeEnabled` |
| 7   | `instructions/admin_grant.rs` | 17-67行目 | `AdminGrantTokenInstruction` |
| 8   | `instructions/emergency_withdraw.rs` | 15-47行目 | `EmergencyWithdraw` |
| 9   | `instructions/deposit.rs` | 15-42行目 | `Deposit` |

---

## ✅ 4. コードレビュー結果（Code Review Result）

| チェック項目                               | 結果（〇安全/×問題あり） |
|--------------------------------------------|--------------------------|
| Accounts構造体に署名者検証があるか         | 〇 安全                 |
| authorityフィールドと署名者の照合を行ったか | 〇 安全                 |
| PDAの場合、signer_seedsの適切な使用        | 〇 安全                 |
| 管理者権限の適切な検証                     | 〇 安全                 |
| ユーザー所有のアカウントへのアクセス制御   | 〇 安全                 |

**詳細な検証結果:**

1. **管理者権限の検証**
   - `admin_fn.rs`の`SetAdmin`と`SetStakeEnabled`構造体では、`#[account(address = staking_pool.admin @ ErrorCode::Unauthorized)]`を使用して管理者の署名を適切に検証しています。
   - `admin_grant.rs`の`AdminGrantTokenInstruction`構造体でも同様に管理者の署名を検証しています。
   - `emergency_withdraw.rs`の`EmergencyWithdraw`構造体でも管理者の署名を検証しています。

2. **ユーザー操作の権限検証**
   - `unstake_token.rs`では、`require!(stake_token.owner == ctx.accounts.unstaker.key(), ErrorCode::Unauthorized)`を使用して、ステークトークンの所有者と署名者が一致することを確認しています。
   - `deposit.rs`では、`#[account(address = from.owner @ ErrorCode::OwnerMismatch)]`を使用して、トークンアカウントの所有者と署名者が一致することを確認しています。

3. **PDAの署名検証**
   - PDAを使用した署名（`with_signer`）が適切に実装されています。例えば、`admin_grant.rs`と`emergency_withdraw.rs`では、正しいシードとバンプを使用してPDAの署名を行っています。

---

## ✅ 5. 危険なコードの抜粋（Insecure Code Snippet）

監査の結果、署名者チェックの欠如に関する重大な問題は発見されませんでした。プログラム全体で適切な署名者検証が実装されています。

ただし、`stake_token.rs`の`StakeTokenInstruction`構造体では、`stake_token`の所有者と`staker`の関連付けが明示的に行われていない点が注意点として挙げられます：

```rust
#[derive(Accounts)]
pub struct StakeTokenInstruction<'info> {
    // ...
    
    /// Stake token PDA
    #[account(
        init_if_needed,
        seeds = [
            b"StakeToken".as_ref(),
            staker.key().to_bytes().as_ref(),
            &staking_pool.version.to_le_bytes(),
            staking_pool.key().to_bytes().as_ref()
        ],
        bump,
        space = StakeTokenAccount::LEN,
        payer = staker
    )]
    pub stake_token: Account<'info, StakeTokenAccount>,
    
    // ...
    
    /// Who is staking the tokens.
    #[account(mut)]
    pub staker: Signer<'info>,
    
    // ...
}
```

しかし、この点については`handle_stake_token`関数内で`stake_token.owner = ctx.accounts.staker.key()`を設定しているため、実質的なセキュリティリスクはありません。

---

## ✅ 6. 影響度評価（Impact Assessment）

- **重要度:** Low（低）
- **潜在的影響:** 署名者チェックは全体的に適切に実装されており、重大なセキュリティリスクは発見されませんでした。

---

## ✅ 7. 推奨する修正方法（Recommended Fix）

現状のコードは署名者チェックの観点から安全に実装されていますが、より明示的な関連付けを行うために、以下の改善を推奨します：

`stake_token.rs`の`StakeTokenInstruction`構造体において、`init_if_needed`を使用している場合は、既存のアカウントに対する所有者チェックを追加することが望ましいです：

```rust
#[derive(Accounts)]
pub struct StakeTokenInstruction<'info> {
    // ...
    
    /// Stake token PDA
    #[account(
        init_if_needed,
        seeds = [
            b"StakeToken".as_ref(),
            staker.key().to_bytes().as_ref(),
            &staking_pool.version.to_le_bytes(),
            staking_pool.key().to_bytes().as_ref()
        ],
        bump,
        space = StakeTokenAccount::LEN,
        payer = staker,
        constraint = !stake_token.initialized || stake_token.owner == staker.key()
    )]
    pub stake_token: Account<'info, StakeTokenAccount>,
    
    // ...
}
```

この変更により、既に初期化されている`stake_token`アカウントに対して、その所有者が現在の`staker`と一致することを明示的に検証します。

---

## ✅ 8. 監査者コメント（Auditor's Comments）

- プログラム全体で署名者チェックが適切に実装されており、セキュリティの観点から良好な状態です。
- 管理者権限を持つ操作（`admin_fn.rs`, `admin_grant.rs`, `emergency_withdraw.rs`）では、特に厳格な署名者検証が行われています。
- ユーザー操作（`stake_token.rs`, `unstake_token.rs`, `deposit.rs`）においても、適切な署名者検証が実装されています。
- PDAを使用した署名も正しく実装されています。

---

## ✅ 9. 再監査の推奨（Recommendation for Re-Audit）

- 現状のコードは署名者チェックの観点から安全に実装されているため、再監査は必要ありません。
- ただし、推奨した改善点を実装する場合は、変更箇所の確認を推奨します。

---

## ✅ 10. 監査チェックリストの記入（Audit Checklist Completion）

| チェック項目                                | チェック結果（✓完了／×未完了） |
|---------------------------------------------|---------------------------------|
| Accounts構造体に署名者チェックがあること    | ✓ 完了                          |
| authorityとSignerのPubkey照合チェック       | ✓ 完了                          |
| PDAの権限チェック（該当する場合）           | ✓ 完了                          |
| 管理者権限の適切な検証                      | ✓ 完了                          |
| ユーザー所有のアカウントへのアクセス制御    | ✓ 完了                          |
| その他関連箇所の権限チェック（コード内全体）| ✓ 完了                          |

> 監査は完了し、すべてのチェック項目を確認しました。プログラム全体で署名者チェックが適切に実装されています。

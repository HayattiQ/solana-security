# オラクル操作（Oracle Manipulation）

## 概要

プログラムが外部オラクル（例：Pyth、Switchboardなど）からデータを取得している場合、オラクルそのものが攻撃対象となる可能性があります。攻撃者がオラクルデータを操作すれば、プログラムが誤った判断を下すリスクがあります。

## 対処法

1. 複数の信頼できるオラクルソースを用いてデータの冗長性・信頼性を確保する
2. オラクルから得たデータが想定範囲内かどうかを必ず検証するロジックを導入する
3. 急激な価格変動などの異常値を検出し、対応するメカニズムを実装する

## 危険なコード例

以下の例では、単一のオラクルからの価格データをそのまま信頼して使用しています：

```rust
// 不正: 単一オラクルからのデータを検証なしで使用
pub fn execute_trade(ctx: Context<ExecuteTrade>, amount: u64) -> Result<()> {
    // オラクルから価格を取得
    let price = ctx.accounts.price_feed.get_price()?;
    
    // 価格を検証せずに取引を実行
    let trade_value = amount * price;
    // ... 取引ロジック ...
    
    Ok(())
}
```

## 安全なコード例

以下の例では、複数のオラクルを使用し、データの検証と異常値の検出を行っています：

```rust
// 安全: 複数オラクルの使用と価格検証
pub fn execute_trade_secure(ctx: Context<ExecuteTradeSecure>, amount: u64) -> Result<()> {
    // 複数のオラクルから価格を取得
    let price1 = ctx.accounts.price_feed1.get_price()?;
    let price2 = ctx.accounts.price_feed2.get_price()?;
    
    // 価格の差異をチェック
    let price_diff_percent = calculate_difference_percent(price1, price2);
    require!(price_diff_percent <= MAX_PRICE_DIFFERENCE, ErrorCode::PriceDiscrepancy);
    
    // 価格が妥当な範囲内かチェック
    let avg_price = (price1 + price2) / 2;
    require!(avg_price >= MIN_ACCEPTABLE_PRICE, ErrorCode::PriceTooLow);
    require!(avg_price <= MAX_ACCEPTABLE_PRICE, ErrorCode::PriceTooHigh);
    
    // 検証済みの価格で取引を実行
    let trade_value = amount * avg_price;
    // ... 取引ロジック ...
    
    Ok(())
}
```

## 追加情報

### オラクル操作攻撃の種類

オラクル操作攻撃には、以下のような種類があります：

1. **フラッシュローン攻撃**
   - 大量の資金を一時的に借り入れ、市場を操作してオラクルの価格に影響を与える
   - 操作された価格を利用して利益を得た後、借入金を返済する

2. **タイムスタンプ操作**
   - 古いデータや遅延したデータを利用して、現在の市場状況と一致しない価格を使用させる
   - 特に更新頻度の低いオラクルで発生しやすい

3. **スラッシング攻撃**
   - オラクルノードやデータ提供者を攻撃して、誤ったデータを報告させる
   - 分散型オラクルシステムの一部のノードを制御することで、コンセンサスに影響を与える

### 安全なオラクル利用のベストプラクティス

1. **複数のオラクルソースの使用**

   単一障害点を排除するために、複数の独立したオラクルからデータを取得します：

   ```rust
   // 複数オラクルの使用例
   #[derive(Accounts)]
   pub struct MultiOraclePrice<'info> {
       pub pyth_price_feed: Account<'info, PythPriceFeed>,
       pub switchboard_price_feed: Account<'info, SwitchboardFeed>,
       pub chainlink_price_feed: Account<'info, ChainlinkFeed>,
   }
   
   pub fn get_validated_price(ctx: Context<MultiOraclePrice>) -> Result<u64> {
       let price1 = ctx.accounts.pyth_price_feed.get_price()?;
       let price2 = ctx.accounts.switchboard_price_feed.get_price()?;
       let price3 = ctx.accounts.chainlink_price_feed.get_price()?;
       
       // 中央値を使用（異常値に強い）
       let median_price = median(price1, price2, price3);
       
       Ok(median_price)
   }
   ```

2. **データの鮮度確認**

   オラクルデータが最新であることを確認します：

   ```rust
   // データの鮮度確認
   pub fn check_price_freshness(price_feed: &PriceFeed) -> Result<()> {
       let current_slot = Clock::get()?.slot;
       let price_slot = price_feed.last_update_slot;
       
       // 一定スロット数以上古いデータは拒否
       require!(
           current_slot - price_slot <= MAX_STALENESS_SLOTS,
           ErrorCode::StaleOracleData
       );
       
       Ok(())
   }
   ```

3. **信頼度スコアの確認**

   多くのオラクルは、データの信頼度や精度に関する情報を提供しています：

   ```rust
   // 信頼度スコアの確認
   pub fn validate_price_confidence(price_feed: &PythPriceFeed) -> Result<()> {
       let confidence = price_feed.confidence;
       let price = price_feed.price;
       
       // 信頼度が価格の一定割合以内であることを確認
       let confidence_ratio = confidence * 100 / price;
       require!(
           confidence_ratio <= MAX_CONFIDENCE_RATIO,
           ErrorCode::LowConfidenceOracleData
       );
       
       Ok(())
   }
   ```

4. **異常値検出と対応**

   急激な価格変動や異常値を検出し、適切に対応します：

   ```rust
   // 異常値検出
   pub fn detect_price_anomaly(
       current_price: u64,
       previous_price: u64,
       time_delta: u64
   ) -> Result<()> {
       // 価格変動率の計算
       let price_change_percent = abs_difference(current_price, previous_price) * 100 / previous_price;
       
       // 単位時間あたりの変動率
       let normalized_change = price_change_percent / time_delta;
       
       // 急激な変動を検出
       if normalized_change > MAX_PRICE_CHANGE_RATE {
           // 対応策：
           // 1. 取引を一時停止
           // 2. より保守的な価格を使用
           // 3. 追加の検証を要求
           return Err(ErrorCode::PriceVolatilityTooHigh.into());
       }
       
       Ok(())
   }
   ```

5. **サーキットブレーカーの実装**

   極端な状況では取引を一時停止するメカニズムを導入します：

   ```rust
   // サーキットブレーカー
   pub fn check_circuit_breaker(ctx: Context<CheckCircuitBreaker>) -> Result<()> {
       let state = &mut ctx.accounts.protocol_state;
       
       // 異常な市場状況を検出
       if is_market_volatile(ctx.accounts.price_feeds) {
           // プロトコルを一時停止
           state.paused = true;
           state.pause_reason = PauseReason::MarketVolatility;
           state.pause_timestamp = Clock::get()?.unix_timestamp;
           
           return Err(ErrorCode::CircuitBreakerTriggered.into());
       }
       
       Ok(())
   }
   ```

### Solanaの主要なオラクルサービス

Solanaエコシステムには、以下のような主要なオラクルサービスがあります：

1. **Pyth Network**
   - 高頻度の価格更新
   - 信頼度メトリクスの提供
   - 複数の取引所からのデータ集約

2. **Switchboard**
   - カスタマイズ可能なオラクルフィード
   - 分散型のデータ提供者ネットワーク
   - 多様なデータソースのサポート

3. **Chainlink**
   - 広範なデータプロバイダーネットワーク
   - 堅牢な検証メカニズム
   - クロスチェーンでの実績

これらのオラクルを使用する際は、それぞれの特性を理解し、適切な検証メカニズムを実装することが重要です。オラクル操作攻撃は、DeFiプロトコルにおける主要な脆弱性の一つであり、適切な対策を講じることで、プロトコルの安全性と信頼性を大幅に向上させることができます。

# keiba_systems


目的：

Goでマイクロサービス（馬券購入〜払い戻し）を作り、TerraformでAWSに構築し、SRE観点で「信頼性・スケーラビリティ・運用性」を学ぶ。

アーキテクチャ構成

┌─────────────────────────────────────────┐
│          API Gateway / Load Balancer    │
└────┬────┬────┬─────┬──────┬───────┬────┘
     │    │    │     │      │       │
  ┌──▼──┐ │ ┌──▼───┐ │  ┌───▼────┐ │
  │馬券  │ │ │馬券  │ │  │オッズ  │ │
  │購入  │ │ │参照  │ │  │参照   │ │
  │API  │ │ │API  │ │  │API    │ │
  └──┬──┘ │ └──┬──┘ │  └───┬───┘ │
     │    │    │    │      │      │
     │  ┌─▼────▼────▼──────▼──────▼──┐
     │  │   レース管理サービス          │
     │  └──────────────────────────────┘
     │                              │
  ┌──▼────┐                    ┌───▼──────┐
  │Message│                    │払い戻し   │
  │Queue  │                    │計算API   │
  │(Kafka)│                    └───┬──────┘
  └──┬────┘                        │
     │                             │
  ┌──▼──────────────┐          ┌───▼──────┐
  │ Redis Writer     │──────────│払い戻し  │
  │ (Consumer)       │          │実行API   │
  └──┬──────────────┘          └───┬──────┘
     │                             │
  ┌──▼────────────────┐            │
  │   Redis Cluster    │ ← Source of Truth
  │ (馬券・オッズ・結果) │◄───────────┘
  └──┬────────────────┘
     │ 非同期バックアップ
     │ (Batch or CDC)
  ┌──▼────────────────┐
  │   PostgreSQL       │  ← 永続化・履歴分析用
  │ (履歴・監査・分析) │
  └───────────────────┘

```



### 2.2 データフロー（改訂版）



#### 馬券購入フロー（MQ統合版）

```

1. ユーザー → 馬券購入API
2. レース状態確認（Redis: race:{race_id}:status == "OPEN"）
3. 購入リクエストをKafkaトピックに送信
4. メッセージID（purchase_id）をユーザーへ即時返却
   ※この時点でRedis未書き込み

--- 非同期処理 ---

5. Redis Writer (Consumer) がKafkaメッセージを受信
6. Redisに書き込み:
   Key: race:{race_id}:tickets:{purchase_id}
   Value:
   {
     "user_id": 12345,
     "horse_number": 5,
     "bet_type": "WIN",
     "amount": 1000,
     "timestamp": "2025-11-03T12:00:00Z"
   }
7. Redisのオッズ（race:{race_id}:odds）を更新
8. 処理結果をpurchase:{purchase_id}:status に保存
   "COMPLETED" or "FAILED"

--- 非同期処理 ---

5. DB書き込みAPI（Consumer）がMQからメッセージを取得
6. PostgreSQLへ馬券データ書き込み
7. 書き込み成功後、Redisのオッズを更新
8. 書き込み結果をステータス管理用Redisに保存

   Key: purchase:{message_id}:status

   Value: "COMPLETED" or "FAILED"

```



#### 購入確認フロー（新規）

```

1. ユーザー → 馬券参照API
2. purchase:{message_id}:status をRedisで確認
3. COMPLETEDなら馬券詳細を返却、PENDINGなら処理中を通知

```



#### オッズ参照フロー（変更なし）

```

1. ユーザー → オッズ参照API
2. Redisから参考オッズ取得
3. ユーザーへ返却

```


#### 締切・確定オッズ計算フロー（変更なし）

```

1. レース終了時刻到達
2. レース状態を "CLOSED" に変更 (race:{race_id}:status)
3. Redisから全馬券情報をスキャン
4. 購入数に基づいて確定オッズを計算
5. Redisキー: race:{race_id}:odds_final に保存
6. PostgreSQLへ結果を非同期バックアップ

```



#### 払い戻し計算フロー（新規）

```

1. レース結果確定
2. 払い戻し計算API起動（イベント駆動 or 定期ジョブ）
3. Redisから対象レースの馬券を抽出
4. 的中馬券をフィルタリングし、確定オッズを参照
5. 払い戻し額を算出し、Redisに保存:
   payout:{race_id}:{user_id} = amount
6. PostgreSQLへ結果をバッチで永続化

```



#### 払い戻し実行フロー（新規）

```

1. ユーザー → 払い戻し実行API
2. Redisキー: payout:{race_id}:{user_id} を確認
3. ステータスをチェックし、未払いなら残高へ反映
4. 冪等性確保のため status:payout:{race_id}:{user_id} = "PAID" を設定

```



## Kafkaメッセージ設計

### トピック構成

topic: betting.ticket.purchase
partition: race_id % partition_count  // レース単位で順序保証

{
  "message_id": "uuid-xxx",
  "user_id": 12345,
  "race_id": 100,
  "horse_number": 5,
  "bet_type": "WIN",
  "amount": 1000,
  "timestamp": "2025-11-03T12:00:00Z"
}

## Redisキー設計

| キー                                     | 用途                  | 型      |
| -------------------------------------- | ------------------- | ------ |
| `race:{race_id}:status`                | レースの状態（OPEN/CLOSED） | String |
| `race:{race_id}:odds`                  | リアルタイムオッズ           | Hash   |
| `race:{race_id}:tickets:{purchase_id}` | 馬券情報                | Hash   |
| `purchase:{purchase_id}:status`        | 馬券処理状態              | String |
| `payout:{race_id}:{user_id}`           | 払い戻し結果              | String |
| `status:payout:{race_id}:{user_id}`    | 払い戻し済フラグ            | String |


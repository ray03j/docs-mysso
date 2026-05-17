# マイクロサービス（Microservices）の通信方式

## 目次

### 同期通信

- [HTTP/REST](#httprest)
- [gRPC](#grpc)
- [GraphQL](#graphql)

### 非同期通信

- [メッセージキュー](#メッセージキューrabbitmq--amazon-sqs-など)
- [イベントストリーミング](#イベントストリーミングkafka--aws-kinesis)
- [Pub/Sub](#pubsubパブリッシュサブスクライブ)

### 基盤技術

- [サービスメッシュ](#サービスメッシュistio--linkerd)
- [APIゲートウェイ](#apiゲートウェイkong--nginx--envoy)
- [サーキットブレイカー](#サーキットブレイカーresilience4j--hystrix)
- [分散トレーシング](#分散トレーシングjaeger--zipkin)

### その他

- [特徴](#特徴)

---

## 各通信方式の詳細

### HTTP/REST

最も一般的。JSON/XMLをやり取り。シンプルで広く普及。

**実装例（Go + net/http）**

```go
// サーバ側
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    userID := r.URL.Query().Get("id")
    user, err := db.GetUser(userID)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(user)
}
http.HandleFunc("/users", getUserHandler)

// クライアント側
resp, err := http.Get("http://user-service:8080/users?id=u123")
```

**メリット**
- 人間が読みやすく、ブラウザやcurlで簡単にデバッグできる。
- 言語やフレームワークに依存しない最も汎用的なプロトコル。
- キャッシュ制御（Cache-Control, ETag）や認証（OAuth, JWT）のエコシステムが豊富。

**デメリット**
- テキストベース（JSON）のシリアライズ/デシリアライズにCPUコストがかかる。
- HTTP/1.1ではコネクションが次々に張られるため、高頻度通信でオーバーヘッドが大きい。
- インターフェースの型安全性が無く、実行時まで不整合に気づけない。

---

### gRPC

HTTP/2 + Protocol Buffers。高性能・型安全。バイナリ通信。

**実装例（.proto + Go）**

```protobuf
syntax = "proto3";
service InventoryService {
  rpc CheckStock (CheckRequest) returns (CheckResponse);
}
message CheckRequest { repeated string item_ids = 1; }
message CheckResponse { bool available = 1; }
```

```go
// サーバ側
func (s *server) CheckStock(ctx context.Context, req *pb.CheckRequest) (*pb.CheckResponse, error) {
    available := s.store.Check(req.ItemIds)
    return &pb.CheckResponse{Available: available}, nil
}

// クライアント側
conn, _ := grpc.Dial("inventory-service:50051", grpc.WithInsecure())
client := pb.NewInventoryServiceClient(conn)
resp, err := client.CheckStock(ctx, &pb.CheckRequest{ItemIds: []string{"i1", "i2"}})
```

**メリット**
- HTTP/2 + バイナリ（Protobuf）により、JSONより高速でペイロードも小さい。
- `.proto` でインターフェースを定義するため、クライアント/サーバ双方で型安全なコードを自動生成できる。
- 双方向ストリーミングを標準でサポート。

**デメリット**
- ブラウザから直接呼び出せない（gRPC-Webやゲートウェイが必要）。
- 読み取り専用のテキストフォーマットではないため、人間による直接デバッグが困難。
- ロードバランシングでHTTP/2の長期コネクションを考慮する必要がある。

---

### GraphQL

クライアントが必要なデータ構造を指定して取得。1回のリクエストで複数リソースにアクセス可能。

**実装例（Apollo Server / Node.js 概念）**

```graphql
type Query {
  user(id: ID!): User
}
type User {
  id: ID!
  name: String
  orders: [Order]
}
```

```javascript
// フロントエンドが必要なフィールドだけを1リクエストで取得
const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      orders { total }
    }
  }
`;
```

**メリット**
- クライアントが必要なデータ構造を指定できるため、オーバーフェッチ/アンダーフェッチを防げる。
- 1回のリクエストで複数のマイクロサービス（ユーザ、注文など）にまたがるデータを集約できる。

**デメリット**
- クエリの複雑さに応じてサーバ負荷が急激に増大しやすい（N+1問題、深いネスト）。
- キャッシュ戦略がRESTより複雑（HTTPレイヤーのキャッシュが使いにくい）。
- 学習コストとスキーマ設計の初期工数が高い。

---

### メッセージキュー（RabbitMQ / Amazon SQS など）

ポイントツーポイント。送信者がキューにメッセージを入れ、受信者が順次処理。

**実装例（Python + pika）**

```python
import pika

# 送信側
connection = pika.BlockingConnection(pika.URLParameters("amqp://rabbitmq"))
channel = connection.channel()
channel.queue_declare(queue="order_queue")
channel.basic_publish(exchange="", routing_key="order_queue", body='{"order_id":"o1"}')

# 受信側
def callback(ch, method, properties, body):
    print(f"Received {body}")
    # 処理後に明示的にACK
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue="order_queue", on_message_callback=callback)
channel.start_consuming()
```

**メリット**
- 受信側が一時的に停止していても、キューがメッセージを保持するためメッセージは失われない。
- 送信側は受信側の処理速度に影響されず、即座に応答を返せる（非同期化によるスループット向上）。
- ポイントツーポイントなので、1メッセージが1受信者に確実に届く（負荷分散しやすい）。

**デメリット**
- 即時性が失われ、リアルタイム更新には向かない。
- メッセージの順序保証や重複排除を考慮する必要がある。
- ブローカーの運用・監視・可用性確保のコストがかかる。

---

### イベントストリーミング（Kafka / AWS Kinesis）

ログのようなストリームにイベントを書き込み、複数の消費者が購読。

**実装例（Kafka / Python）**

```python
from kafka import KafkaProducer, KafkaConsumer

# 送信側（プロデューサ）
producer = KafkaProducer(
    bootstrap_servers=["kafka:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8")
)
producer.send("orders", value={"order_id": "o1", "status": "created"})

# 受信側（コンシューマ）
consumer = KafkaConsumer("orders", bootstrap_servers=["kafka:9092"])
for msg in consumer:
    process(msg.value)
```

**メリット**
- イベントを時系列ログとして保持するため、過去のイベントを「巻き戻して再処理」できる。
- 複数の独立した消費者グループが同じストリームを並列に購読できる。
- 高いスループットと耐久性を持つ。

**デメリット**
- 「キュー」とは異なり、1メッセージが複数消費者に届くため、冪等性設計が必須。
- Kafkaなどは運用が複雑（ZooKeeper/KRaft、パーティション管理、レプリケーションなど）。
- 消費者の遅延（Lag）を監視し、適切にスケールする運用負荷がある。

---

### Pub/Sub（パブリッシュ/サブスクライブ）

トピックベースの配信。1:nの非同期通信。

**実装例（Google Cloud Pub/Sub / Python）**

```python
from google.cloud import pubsub_v1

# 送信側
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "order-events")
publisher.publish(topic_path, b'{"event": "order_placed"}')

# 受信側
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "order-sub")
def callback(message):
    print(message.data)
    message.ack()
subscriber.subscribe(subscription_path, callback=callback)
```

**メリット**
- 送信側は「誰が受け取るか」を意識せず、トピックに投げるだけでよい（完全な疎結合）。
- クラウドマネージドサービスを使えば運用負荷が大幅に減る。
- 動的に購読者を追加・削除できるため、スケーリングが柔軟。

**デメリット**
- メッセージの順序保証が難しい（特にクラウドPub/Subは順序保証に制限がある）。
- 同様に冪等性設計が必須。
- メッセージの保持期間やデッドレターキューの管理が必要。

---

### サービスメッシュ（Istio / Linkerd）

サービス間通信をサイドカープロキシで抽象化。ロードバランシング、暗号化、認証、トレースをアプリケーションコードから分離。

**実装例（Istio / Kubernetes 概念）**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-route
spec:
  hosts:
  - inventory-service
  http:
  - route:
    - destination:
        host: inventory-service
    timeout: 2s
    retries:
      attempts: 3
      perTryTimeout: 1s
```

**メリット**
- アプリケーションコードを変更せずに、暗号化（mTLS）、認証、レート制限、トレースを追加できる。
- サイドカー（プロキシ）が通信を横取りするため、言語やフレームワークに依存しない。
- カナリアリリースやA/Bテストを宣言的に設定できる。

**デメリット**
- サイドカーの追加により、レイテンシが増加し、メモリ/CPUリソースが増える。
- 全体のネットワークトポロジーが複雑化し、トラブルシューティングが困難になる。
- Istioなどは学習コストと運用コストが高い。

---

### APIゲートウェイ（Kong / NGINX / Envoy）

クライアントからの入口を一元化。認証、レート制限、ルーティング、プロトコル変換を担当。

**実装例（Kong / declarative config 概念）**

```yaml
services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - paths: ["/users"]
plugins:
  - name: rate-limiting
    config:
      minute: 100
  - name: jwt
```

**メリット**
- クライアントが各マイクロサービスのエンドポイントを直接知る必要がなく、入口が1つにまとまる。
- 共通の横断的関心事（認証、レート制限、CORS、ログ）をアプリから分離できる。
- プロトコル変換（REST ↔ gRPC）をゲートウェイで吸収できる。

**デメリット**
- ゲートウェイが単一障害点（SPOF）になりうるため、可用性設計が必要。
- すべての通信がゲートウェイを経由するため、ボトルネックになりやすい。
- 設定の複雑化により、ルーティングの管理工数が増える。

---

### サーキットブレイカー（Resilience4j / Hystrix）

連鎖的な障害を防ぐ。下流サービスの障害時に即座に失敗を返す。

**実装例（Java / Resilience4j）**

```java
CircuitBreaker cb = CircuitBreaker.ofDefaults("inventory");

Supplier<String> decorated = CircuitBreaker
    .decorateSupplier(cb, () -> restTemplate.getForObject(url, String.class));

Try<String> result = Try.ofSupplier(decorated);
if (result.isFailure()) {
    return fallbackResponse();
}
```

**メリット**
- 下流サービスの障害が連鎖的に上流に波及するのを防ぐ。
- 回復時間を与えるため、一時的な過負荷からの自動回復が期待できる。

**デメリット**
- 設定（失敗率の閾値、待機時間など）のチューニングが難しい。緩すぎると意味がなく、厳しすぎると正常時にも開いてしまう。
- フォールバック（代替処理）の設計自体に工数がかかる。
- 分散システムの動作を直感的に把握しにくくなる。

---

### 分散トレーシング（Jaeger / Zipkin）

複数サービスにまたがるリクエストの流れを追跡。

**実装例（OpenTelemetry / Go 概念）**

```go
ctx, span := tracer.Start(ctx, "place-order")
defer span.End()

// 別サービス呼び出し時にコンテキストを伝播
resp, err := inventoryClient.CheckStock(ctx, req)
```

**メリット**
- 複数サービスにまたがる1リクエストの流れを可視化し、ボトルネックの特定が容易になる。
- ログだけでは分からない「どのサービス間で時間がかかっているか」を把握できる。

**デメリット**
- スパン情報の収集・保存にリソースが必要（特に高頻度通信時）。
- すべてのサービスに計装（Instrumentation）を導入する工数がかかる。
- トレースIDの伝播忘れや中断により、途切れたトレースが生じやすい。

---

## 特徴

- 通信はネットワーク越しになるため、レイテンシが増大し、失敗の可能性が常にある。
- **「分散トランザクション」**は避けるべき原則（Sagaパターンや最終的整合性を採用）。
- プロトコルとインターフェースの厳格な定義（スキーマ、バージョニング）が必須。

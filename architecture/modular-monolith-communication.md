# モジュラーモノリス（Modular Monolith）の通信方式

## 目次

- [公開API経由のメソッド呼び出し](#公開api経由のメソッド呼び出し)
- [依存性注入（DI）](#依存性注入di)
- [ドメインイベント（インメモリ）](#ドメインイベントインメモリ)
- [トランザクショナルアウトボックス](#トランザクショナルアウトボックス)

---

## 各通信方式の詳細

### 公開API経由のメソッド呼び出し

各モジュールが公開インターフェースを提供し、他モジュールはそれ経由で呼び出す。内部実装は隠蔽される。

**実装例（Go）**

```go
// billing/api.go （公開API）
package billing

type API interface {
    CreateInvoice(orderID string, amount int) error
}

type api struct{ /* 非公開実装 */ }

func NewAPI() API { return &api{} }
func (a *api) CreateInvoice(orderID string, amount int) error { /* ... */ return nil }

// order/module.go
package order

type Service struct {
    billingAPI billing.API // 公開APIのみに依存
}

func (s *Service) PlaceOrder() {
    s.billingAPI.CreateInvoice("order-123", 1000)
}
```

**メリット**
- モジュール内部の変更が、公開APIを保つ限り他モジュールに影響しない。
- コードレビューで「直接DBテーブルを参照していないか」などの境界違反を検出しやすい。

**デメリット**
- インターフェース設計に初期コストがかかる。
- プロセス内通信なのに「公開/非公開」を意識するため、単純な関数呼び出しより記述量が増える。

---

### 依存性注入（DI）

インターフェース経由で他モジュールの機能を利用。実装の結合度を下げる。

**実装例（Python + 手動DI）**

```python
class BillingAPI:
    def create_invoice(self, order_id: str, amount: int): ...

class OrderService:
    def __init__(self, billing_api: BillingAPI):
        self._billing_api = billing_api

    def place_order(self, items: list):
        # 具象クラスではなく、注入されたインターフェースに依存
        self._billing_api.create_invoice("order-123", sum(i.price for i in items))

# 実行時に注入
service = OrderService(billing_api=BillingAPI())
```

**メリット**
- テスト時にモック実装を注入でき、ユニットテストが容易になる。
- モジュール間の依存関係がコンストラクタで明示的になり、全体像を把握しやすい。

**デメリット**
- DIコンテナやファクトリの導入・学習コストがかかる場合がある。
- 過度に抽象化すると、実際の動作を追跡するのが難しくなる（「new しない」ことによる可読性低下）。

---

### ドメインイベント（インメモリ）

同一プロセス内のインメモリイベントバスを使い、モジュール間の疎結合を実現。

**実装例（Go）**

```go
package eventbus

type Handler func(event DomainEvent)
var subscribers = map[string][]Handler{}

func Subscribe(eventType string, h Handler) {
    subscribers[eventType] = append(subscribers[eventType], h)
}
func Publish(e DomainEvent) {
    for _, h := range subscribers[e.Type] { h(e) }
}

// --- 別モジュール ---
// inventory/module.go
func init() {
    eventbus.Subscribe("order.placed", func(e eventbus.DomainEvent) {
        decreaseStock(e.Payload["items"])
    })
}
```

**メリット**
- モジュール間が完全に疎結合になり、互いの存在を知らなくても連携できる。
- 将来メッセージブローカーに差し替える際、イベント定義の変更が少なくて済む。

**デメリット**
- イベントは即座に処理されるため、リトライや冪等性の担保がインメモリでは困難。
- イベントハンドラ内で例外が発生すると、発信元のトランザクション全体が失敗する（連鎖障害）。

---

### トランザクショナルアウトボックス

将来のマイクロサービス化を見越し、DBにイベントを書き込んでから別プロセスやバッチに配信する準備をする場合がある。

**実装例（概念 / SQL）**

```sql
-- アプリケーションDB内にアウトボックステーブルを持つ
CREATE TABLE outbox (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP
);

-- ビジネストランザクションと同一トランザクションで書き込む
BEGIN;
INSERT INTO orders (user_id, total) VALUES ('u123', 1000);
INSERT INTO outbox (event_type, payload)
VALUES ('order.created', '{"order_id": 1, "user_id": "u123"}');
COMMIT;

-- 別プロセス（ポーラー）が未処理レコードを読み出して配信
SELECT * FROM outbox WHERE processed_at IS NULL ORDER BY id;
```

**メリット**
- 「ビジネスDB更新」と「イベント発行」を同一トランザクションで担保できるため、データ不整合を防ぐ。
- 将来メッセージブローカー（Kafkaなど）に差し替えてマイクロサービス化する際の移行パスになる。

**デメリット**
- ポーラーによる配信遅延が生じるため、即時性が失われる。
- アウトボックステーブルの管理（古いレコードの削除、重複配信の防止）が必要。

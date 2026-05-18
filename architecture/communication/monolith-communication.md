# モノリス（Monolith）の通信方式

## 目次

- [メソッド/関数呼び出し](#メソッド関数呼び出し)
- [共有メモリ](#共有メモリ)
- [インメモリイベントバス](#インメモリイベントバス)
- [グローバル状態/シングルトン](#グローバル状態シングルトン)

---

## 各通信方式の詳細

### メソッド/関数呼び出し

同一コードベース内での直接的な呼び出し。最も一般的なプロセス内通信方式。

**実装例（Python）**

```python
class OrderService:
    def create_order(self, user_id: str, items: list):
        # 在庫モジュールの関数を直接呼び出す
        inventory_service = InventoryService()
        if not inventory_service.check_stock(items):
            raise ValueError("在庫不足")
        # 注文処理...
```

**メリット**
- 呼び出しコストがほぼゼロ（スタックジャンプのみ）。
- 型安全やIDEの補完が効きやすく、開発効率が高い。
- トランザクション管理が同一DB接続内で完結しやすい。

**デメリット**
- 呼び出し側と被呼び出し側が密結合になり、一方の変更が他方に影響しやすい。
- 内部実装の詳細が露出し、カプセル化が破られることが多い。
- スレッドブロッキングが連鎖しやすく、障害の影響範囲が広がる。

---

### 共有メモリ

スレッド間やクラス間でメモリ空間を共有してデータを受け渡し。

**実装例（Python + threading）**

```python
from threading import Lock

class SharedCache:
    _data = {}
    _lock = Lock()

    @classmethod
    def get(cls, key):
        with cls._lock:
            return cls._data.get(key)

    @classmethod
    def set(cls, key, value):
        with cls._lock:
            cls._data[key] = value
```

**メリット**
- データコピーを行わないため、大量データの受け渡しでもオーバーヘッドが極めて小さい。
- 実装が単純で、追加のライブラリやインフラが不要。

**デメリット**
- スレッドセーフを保証するための排他制御（ロック）が必要で、実装を複雑化させる。
- デッドロックやレースコンディションのリスクが高い。
- 複数プロセス（マイクロサービス化）に移行する際に完全に書き換えが必要。

---

### インメモリイベントバス

同一プロセス内でのみ動作するPub/Sub。Observerパターンなど。

**実装例（簡易Observerパターン / Python）**

```python
class EventBus:
    _subscribers = {}

    @classmethod
    def subscribe(cls, event_type, handler):
        cls._subscribers.setdefault(event_type, []).append(handler)

    @classmethod
    def publish(cls, event_type, payload):
        for handler in cls._subscribers.get(event_type, []):
            handler(payload)

# 利用例
EventBus.subscribe("order_created", lambda e: send_email(e["user_id"]))
EventBus.publish("order_created", {"user_id": "u123", "order_id": "o456"})
```

**メリット**
- 発信側と受信側が疎結合になり、互いの実装を知らなくても連携できる。
- 同一プロセス内でのみ動作するため、ネットワーク遅延やシリアライズコストがない。

**デメリット**
- イベントの処理順序や失敗時の動作（ロールバック、リトライ）を自前で管理する必要がある。
- イベントハンドラ内で例外が発生すると、発信元の処理にも影響を与えやすい。
- デバッグが難しく、イベントの流れを追跡する仕組みが必要。

---

### グローバル状態/シングルトン

共有された状態に直接アクセス（非推奨だが見られる）。

**メリット**
- どこからでも即座にアクセスでき、実装コストが最小。

**デメリット**
- テストが極めて困難（モック化しにくい）。
- 並行アクセス時の競合状態が発生しやすい。
- コードのどこからでも変更可能なため、予期しない副作用を生みやすい。

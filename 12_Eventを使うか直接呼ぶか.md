# 第12章　Eventを使うか、直接呼ぶか

## 1. まず直接呼び出しを考える

ある処理の後に別の処理を実行したい。

例えばJob完了後に通知を送る。

最も直接的には、

```python
def execute(*args, **kwargs):
    complete_job(...)
    send_notification(...)
    connection.commit()
```

と書ける。

この形では、UseCaseが、

> Job完了後に通知する

という処理順序を明示している。

具象設計論では、まずこの直接的な表現を基準にする。

---

## 2. Eventは「疎結合にする魔法」ではない

Eventを導入すると、

```python
publish(
    JobCompleted(...)
)
```

と書ける。

Job完了処理はNotificationを直接呼ばなくなる。

しかし依存が消えたわけではない。

代わりに、

- Event
- Publisher
- Event delivery
- Handler registration
- Handler
- Error handling

という新しい責任が生まれる。

Eventは依存をなくすのではなく、依存の形を変える。

---

## 3. 「能力を必要とする」と「事実を知らせる」を分ける

UseCaseが通知送信の成功を必要としているなら、

```python
send_notification(...)
```

を直接呼ぶ方が意味を表しやすい。

一方、

> Jobが完了したという事実を知らせたい。誰がその事実を利用するかはJob完了処理の責任ではない。

という場合はEventが適する可能性がある。

この違いを、

- 能力を要求する
- 起きた事実を通知する

として分ける。

---

## 4. ProducerがConsumerを知ってよい場合

Complete Job Featureの仕様として、

> 必ず担当者へ通知する

ことが明確なら、UseCaseがNotificationを直接利用してよい。

```python
def execute(*args, **kwargs):
    complete_job(...)
    send_notification(...)
```

Consumerを知っていること自体は問題ではない。

利用目的上、必要な依存だからである。

---

## 5. ProducerがConsumerを知らない方がよい場合

Job Completedをきっかけに、

- 通知
- 集計更新
- 外部連携
- Audit

などが独立して追加されるとする。

Complete Job Featureがそれらすべての処理順序を担う意味が薄い場合、

```python
publish(
    JobCompleted(
        job_id=job_id,
    )
)
```

という事実の公開を検討できる。

ただし、Eventを導入することで全体の追跡性は下がる。

このコストを受け入れる理由が必要である。

---

## 6. Eventは新しい公開面である

Eventを発行することは、

> この事実を他概念から利用してよい

という新しい公開面を作ることでもある。

Event名には、

```text
JobCompleted
ScheduleChanged
OrderAccepted
```

のように、起きた事実を表す名前を付ける。

```text
NotifyUserEvent
UpdateDashboardEvent
```

のようにConsumer側の処理名をEventへ入れると、ProducerがConsumerの目的を知る形へ戻りやすい。

---

## 7. Event Handlerの配置

Handlerも役割である。

例えばJob CompletedをWeb Notification機能が利用するなら、

```text
feature/
    notification/
        job_completed_handler.py
```

のように置ける。

Handlerだからトップレベルの、

```text
event_handlers/
```

を作る必要はない。

そのHandlerが何の概念に属するかで配置を決める。

---

## 8. 同期Event

同一Process内で、

```python
publish(event)
```

した直後にHandlerを同期実行する方式もある。

この場合、

- 呼び出し元はHandlerを直接知らない
- しかし同じProcess内で失敗が伝播する

という性質を持つ。

直接呼び出しより追跡が難しくなる一方、複数Consumerを独立して追加しやすくなる。

この利益が必要かを判断する。

---

## 9. 非同期Event

Queue等を使って非同期配送する場合は、さらに、

- 配送失敗
- Retry
- 重複配送
- 順序
- Event schema
- Consumer version

などの問題が追加される。

非同期Eventは、単なる関数呼び出しの置換ではない。

分散処理という新しい設計問題を導入する。

---

## 10. TransactionとEvent

Database更新とEvent発行の整合性が必要になる。

```python
connection.commit()
publish(event)
```

では、commit後にEvent発行が失敗する可能性がある。

```python
publish(event)
connection.commit()
```

では、Event発行後にcommitが失敗する可能性がある。

必要ならOutbox等を導入する。

Eventを使うことでTransaction問題が解決するわけではない。

---

## 11. Eventを何でも使わない

「疎結合だから」という理由で、

```text
UserLoaded
JobValidated
RepositorySaved
MailRequested
```

と細かな内部処理までEvent化すると、処理順序がコードから見えなくなる。

直接呼び出しで十分な依存は、直接呼ぶ。

Eventは、

> 事実として公開する意味があり、ProducerがConsumerを知らないことに価値がある

場合に使う。

---

## 12. Eventにも標準構成とTestが必要になる

Event Publisherを利用するモジュールでは、それをDependencyとして定義できる。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    publish: Publish

    @classmethod
    def default(cls) -> Self:
        return cls(
            publish=event_publisher,
        )
```

テストでは、

```python
events = []

dependencies = Dependencies(
    publish=events.append,
)
```

として、何を発行したか確認できる。

EventだからDependency設計が別物になるわけではない。

---

## 13. Direct callとEventの判断

次の問いを使える。

### 直接呼ぶ方がよい可能性が高い

- Consumerの成功が利用目的の成功に必要
- 処理順序が明確
- Consumer数が少ない
- 依存関係をコードから追える方が重要

### Eventを検討できる

- 起きた事実自体を公開したい
- ConsumerをProducerが知る意味が薄い
- Consumerが独立して増減する
- 非同期化に具体的な価値がある

Eventを使う条件を完全に決定する規則ではない。

Event導入によって得るものと失うものを比較するための観点である。

---

## 14. この章の判断原則

1. まず直接呼び出しを考える
2. Eventは依存を消すものではない
3. 能力要求と事実通知を分ける
4. ProducerがConsumerを知ることを無条件に悪としない
5. Event名は起きた事実を表す
6. Handlerも意味のある概念の配下へ置く
7. 非同期化の運用コストを含めて判断する
8. Transactionとの不整合を隠さない
9. Event Publisherも通常のDependencyとして扱う

ここまでで、読み取り、入口、外部依存、Eventという境界側の設計を扱った。

次部では、具体例が増えたコードをどのように見直し、概念を分割・統合していくかを扱う。

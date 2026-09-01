# 第8章　Entryと外部境界を配置する

## この章の目的

外部から処理が入るEntryと、Applicationから外部サービスを利用するOutbound境界を分ける。外部表現、Framework、通信制御、Eventを、それぞれの意味と変更理由に応じて配置する。

## 1. EntryはInbound経路を扱う

Entryは、外部からApplicationへ処理が入る経路である。

```text
entry/
    web.py
    cli.py
```

Webが成長したら、同名Packageへ深掘りできる。

```text
entry/
    web/
        __init__.py
        routes.py
        session.py
    cli.py
```

HTTP、CLI、Batch、Message consumerなどはEntryの具体例である。Applicationから外部APIを呼ぶ処理はEntryではなく、Outbound側の境界として扱う。

## 2. 外部表現を内部で使う値へ変換する

```python
from feature.complete_job import execute


def handle(request):
    job_id = int(request.path_params["job_id"])
    reason = request.form["reason"]

    result = execute(job_id=job_id, reason=reason)

    return JSONResponse(
        {
            "job_id": result.job_id,
            "completed_at": result.completed_at.isoformat(),
        }
    )
```

EntryはRequestをFeatureが必要とする値へ変換し、Featureの結果をResponseへ変換する。

整数として解釈できないJob IDはWeb入力の問題として扱える。Completed Jobは変更できないという規則は、Jobの状態規則として扱える。同じValidationという名前でも、変更理由を分ける。

## 3. Entryを薄くすることを目的にしない

次の責任はWebに置く意味がある。

- HeaderやCookieを読む
- Sessionを更新する
- Pagination parameterを解釈する
- Templateを選ぶ
- Featureの失敗をHTTP Responseへ変換する

入口固有の処理まで別層へ押し出して、Controllerの行数だけを減らす必要はない。

重要なのは、外部表現の都合が無関係なFeatureやData Componentへ広がらないことである。

## 4. Framework型を運ぶ範囲を判断する

FastAPIやDjangoのRequest Modelを、FeatureやData Componentへそのまま渡す必要はない。

```python
class CompleteJobRequest(BaseModel):
    job_id: int
    reason: str
```

を、必要ならPython標準の型へ変換できる。

```python
@dataclass(frozen=True, slots=True)
class CompleteJobInput:
    job_id: int
    reason: str
```

変換によってFramework依存の範囲や内部で使う意味が明確になるなら行う。境界ごとにDTOを一式増やすこと自体は目的にしない。

Entry固有のUseCaseなら、Framework型を使う合理的な場合もある。

## 5. 外部サービスはOutbound側の意味として配置する

外部APIやSDKを一度呼ぶだけなら、利用するFeatureの近くに置ける。

一方、Stripeとの連携に次の責任があるとする。

- Authentication
- Request・Response変換
- Timeout
- Retry
- Rate Limit
- SDK固有Errorの変換

これらがStripeとの接続という独立した意味を持つなら、次の配置を作れる。

```text
external_service/
    stripe.py
```

外部技術だから遠くへ置くのではない。外部サービスとの通信と外部都合を扱う責任が具体化したため、名前を与える。

## 6. RetryやTimeoutの意味を分ける

通信の一時失敗、Rate Limit、SDK例外に対するRetryは、外部サービス側の責任になりやすい。

一方、業務として最大三回試行し、失敗後に手動確認状態へ移すなら、その回数と失敗後の処理はFeatureの判断になり得る。

Retryという役割名だけで配置せず、通信条件なのか利用目的の規則なのかを見る。

## 7. まず直接呼び出しを考える

Job完了後の通知がComplete Jobの成功条件なら、UseCaseから直接呼べる。

```python
def execute(...):
    complete_job(...)
    send_notification(...)
    connection.commit()
```

Consumerを知っていること自体は問題ではない。利用目的がその能力を必要としている。

## 8. Eventは事実を公開する必要があるときに使う

Job完了という事実を、通知、集計、Audit、外部連携が独立して利用するならEventを検討できる。

```python
publish(JobCompleted(job_id=job_id))
```

Eventは依存を消さない。Publisher、配送、Handler登録、失敗処理、schema管理などの責任を追加する。

Event名は、`JobCompleted`のように起きた事実を表す。`NotifyUserEvent`のようにConsumerの命令をProducer側のEvent名へ入れると、事実通知と能力要求が混ざりやすい。

Handlerも役割である。通知FeatureがJob Completedを利用するなら、次のように意味の配下へ置ける。

```text
feature/
    notification/
        job_completed_handler.py
```

Handlerという理由だけでトップレベルの`event_handlers/`を作る必要はない。

## 9. 非同期化の物理的な条件を隠さない

Queueを使う非同期Eventでは、配送失敗、Retry、重複、順序、schema version、Consumer versionを扱う必要がある。

Database commitとEvent発行も同じAtomicな処理とは限らない。必要ならOutbox等を追加する。

Eventは直接呼び出しの表記を短くするために導入するものではない。ConsumerをProducerから分離する価値と、追跡性・運用責任の増加を比較する。

## 10. 例: Webから外部通知まで

```text
entry/
    web/
        complete_job.py

feature/
    complete_job.py

data_component/
    job/

external_service/
    mail.py
```

1. Web EntryがRequestを内部値へ変換する
2. Complete Job FeatureがJobの公開操作を利用する
3. Featureの成功条件に通知が含まれるならMail送信を直接利用する
4. Mail固有の認証、Timeout、Error変換は`external_service/mail.py`へ置く
5. 後からJob Completedを複数Consumerへ公開する意味が生じたらEventを再検討する

配置結果だけでなく、どの責任が具体化したために各概念を作ったかを示す。

## 11. この章の結論

EntryはInbound経路、外部サービスはOutbound境界として区別する。外部表現、Framework型、通信制御を、それを必要とする意味の範囲へ置く。

能力の成功が利用目的に必要なら直接呼ぶ。起きた事実を独立したConsumerへ公開する必要があるならEventを検討する。抽象化によって外部I/Oの失敗や非Atomic性を隠さない。

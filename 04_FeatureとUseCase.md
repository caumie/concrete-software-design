# 第4章　FeatureとUseCase

## 1. Featureは利用目的を具体化する

Featureは、利用者またはシステムが達成しようとする一つの目的を、コード上のまとまりとして表す。

例えば、

```text
feature/
    create_job/
    complete_job/
    reschedule_job/
```

とする。

ここで重要なのは、Featureを「Application層の別名」としないことである。

Featureは、

> 何を実現しようとしているか

という具体的な利用目的に名前を与えたものである。

Featureは利用目的をまとめるが、Featureから到達できるすべての変更理由を引き受けるものではない。利用するData Componentや外部サービスの内容が変わったために複数Featureが変更されるなら、その変更の原因は依存先の意味や公開面にある。

Create Job、Complete Job、Reschedule Jobという名前から、その配下に置かれるコードが何のために存在するのかを説明できる。

---

## 2. 最初からFeature内部を揃えない

Featureを作ったからといって、すべてのFeatureに同じファイル構成を要求しない。

単純なFeatureなら、

```text
feature/
    complete_job/
        __init__.py
        usecase.py
```

だけでよい。

必要になったときに、

```text
feature/
    complete_job/
        __init__.py
        usecase.py
        query.py
```

のように役割を増やす。

さらに依存を明示する必要があれば、そのモジュール内で依存定義を持つこともある。

重要なのは、

> Featureを作ったからUseCase、Query、DTO、Factory、Providerを一式作る

という順序にしないことである。

必要な差異が現れたときにだけ役割を増やす。

---

## 3. UseCaseは利用目的の処理順序を表す

Featureの中心には、しばしばUseCaseがある。

Complete Jobを例にする。

Job Data Componentが、

```python
from data_component.job import complete_job
```

という公開機能を提供しているとする。

Feature側はそれを利用する。

```python
# feature/complete_job/usecase.py

from data_component.job import complete_job


def execute(
    job_id: int,
    connection,
) -> None:
    complete_job(
        job_id,
        connection,
    )

    connection.commit()
```

ここでUseCaseが知っているのは、

1. Jobを完了させる
2. その処理が成功したらTransactionを確定する

という利用目的上の処理順序である。

Jobをどのように取得するか、どの状態から完了可能か、どのように保存するかは、Job側の責任として隠すことができる。

---

## 4. UseCaseは「何でもするサービス」ではない

UseCaseを、Featureに必要なすべてのロジックを置く場所として使うと、再び一つの大きな手続きへ戻る。

例えば、

```python
def execute(job_id, connection):
    row = connection.execute(...).fetchone()

    if row["status"] != "running":
        raise ValueError(...)

    connection.execute(...)

    send_notification(...)

    render_html(...)
    connection.commit()
```

とすれば、UseCase一つに、

- 永続化
- Domain Rule
- 通知
- 表示

が混ざる。

UseCaseの役割は、

> 利用目的を成立させるために、どの概念をどの順序で利用するかを決めること

である。

各概念自身が持つべき判断までUseCaseへ引き上げる必要はない。

---

## 5. Feature固有の判断はFeatureに残す

一方、すべての判断をData Componentへ移す必要もない。

例えばComplete Jobでは、利用者に「完了理由」の入力を要求するとする。

```python
def execute(
    job_id: int,
    reason: str,
    connection,
) -> None:
    if not reason.strip():
        raise ValueError("completion reason is required")

    complete_job(
        job_id,
        connection,
    )

    connection.commit()
```

完了理由が必須であるという規則が、

> Jobという状態を持つ概念の普遍的な規則

ではなく、

> Complete Jobという利用目的における要求

であるなら、Feature側に置く方が自然である。

ここでも判断基準は「Domain LogicかApplication Logicか」という分類名ではない。

**何の変更理由に属する判断なのか**を見る。

この判断では、変更されたファイルの数ではなく、何の意味が変わったかを区別する。Complete Job固有の要求が変わったならFeatureの変更であり、Jobの状態規則が変わったならData Componentの変更である。後者の影響で複数Featureの呼び出し方が変わっても、それをFeature横断の規則としてFeature側へ押し込まない。

---

## 6. UseCaseは複数の概念を協調させる

Reschedule Jobでは、JobとScheduleの両方を扱うとする。

```python
from data_component.job import ensure_reschedulable
from data_component.schedule import change_period


def execute(
    job_id: int,
    schedule_id: int,
    period,
    connection,
) -> None:
    ensure_reschedulable(
        job_id,
        connection,
    )

    change_period(
        schedule_id,
        period,
        connection,
    )

    connection.commit()
```

JobとScheduleは別のData Componentである。

それでも、一つの利用目的のために両方を利用してよい。

UseCaseは、複数の概念を一つの処理として協調させる場所になり得る。

このことは、Data Component同士を直接依存させる必要がないことにもつながる。

---

## 7. Data Component同士を直接協調させない

例えばJobがScheduleを直接取得して変更する。

```python
class Job:
    def reschedule(self, schedule_repository, period):
        schedule = schedule_repository.get(self.schedule_id)
        schedule.change_period(period)
        schedule_repository.save(schedule)
```

この形ではJobが、

- Scheduleという別概念
- Schedule Repository
- 処理順序

まで知る。

Job自身の規則と、複数概念を協調させる利用目的が混ざる。

代わりにUseCaseから、

```python
job.ensure_reschedulable()
schedule.change_period(period)
```

と順番を決める。

どのData Componentを組み合わせるかはFeatureの利用目的に属する。

---

## 8. FeatureはEntryを知らなくてもよい

Webから利用するFeatureであっても、Feature自身がHTTP Requestを知る必要がないなら、知らない方がよい。

望ましくない例は、

```python
def execute(request, connection):
    job_id = int(request.path_params["job_id"])
    reason = request.form["reason"]
    ...
```

である。

これではComplete Job FeatureがWebの入力表現に依存する。

Web側で、

```python
def handle(request, connection):
    return execute(
        job_id=int(request.path_params["job_id"]),
        reason=request.form["reason"],
        connection=connection,
    )
```

と具体化できる。

Featureは、自分が必要とする値だけを受け取る。

---

## 9. Entry固有UseCaseはEntryの配下に置ける

UseCaseだから必ず`feature/`に置く、という規則にはしない。

例えば、

```text
entry/
    web/
        restore_session/
            usecase.py
```

がある。

この処理がWeb Sessionにしか意味を持たないのであれば、`entry/web`の配下にあってよい。

反対に、後から同じ処理がCLIや他の入口でも利用され、Webから独立した意味を持つようになったなら、Featureとして切り出すことを検討する。

ここでも判断基準は、

> UseCaseという役割だからどこへ置くか

ではない。

> このUseCaseは何という概念の配下なら説明できるか

である。

---

## 10. QueryもFeatureの内部に置ける

Complete Jobのためだけに補助情報を取得するQueryが必要になったとする。

```text
feature/
    complete_job/
        usecase.py
        query.py
```

このQueryがComplete Jobという利用目的にしか意味を持たないなら、Feature内部へ置く。

Queryという役割だけを理由に、

```text
query/
```

というトップレベル構造を作る必要はない。

Queryについては後の章で詳しく扱うが、Featureとの関係で重要なのは、

> Queryも利用目的の一部になり得る

という点である。

---

## 11. Featureが大きくなったら再び具体例として見る

Feature内部が増えてきたとする。

```text
feature/
    reschedule_job/
        usecase.py
        query.py
        calculation.py
        validation.py
        notification.py
```

この時点で、

> ファイルが五つになったから分割する

という規則はない。

しかし、具体例が増えたことで判断材料は増えている。

`calculation.py` はScheduleという概念の規則ではないか。`notification.py` は別Featureでも同じ意味を持っていないか。`validation.py` はUseCase固有なのか、Data Componentの不変条件なのか。

Featureは完成形ではない。

Feature内部のコードを俯瞰することで、新しい概念を発見することもある。

---

## 12. Featureを統合することもある

逆に、

```text
feature/
    change_job/
    revise_job/
```

が別々に存在していても、具体化した結果、同じ利用目的の異なる呼び方にすぎないと分かることがある。

その場合は統合してよい。

具象設計論は、

> Featureを増やすこと

を目的にしていない。

Featureは現在認識している利用目的を表す仮説である。

意味が変われば、分割も統合もできる。

---

## 13. Featureの公開面

Featureも他概念から利用されるなら、トップで公開面を持てる。

```python
# feature/complete_job/__init__.py

from .usecase import execute

__all__ = ["execute"]
```

Webからは、

```python
from feature.complete_job import execute
```

と利用する。

Feature内部の`query.py`や補助関数をWebが直接利用する必要がなければ、公開しない。

これによって、Feature内部の再構成をFeature外部へ波及させにくくできる。

---

## 14. FeatureとUseCaseの関係を固定しすぎない

多くの場合、

```text
Feature
  └── UseCase
```

という関係になる。

しかし、FeatureとUseCaseを必ず一対一にする必要はない。

非常に小さなFeatureなら、公開関数そのものがUseCaseとして成立する。

逆に、一つの利用目的の中に複数の実行経路が存在し、それぞれに独立した意味があるなら、複数のUseCaseを持つ場合もある。

ただし、UseCaseが増えたことでFeatureの意味が曖昧になるなら、Feature分割を考える材料になる。

---

## 15. この章の判断原則

FeatureとUseCaseについて、具象設計論では次のように考える。

1. Featureは具体的な利用目的に名前を与える
2. Feature内部の役割は必要になったものだけ作る
3. UseCaseは利用目的の処理順序を担う
4. Data Component自身の規則をUseCaseへ引き上げすぎない
5. Feature固有の判断をData Componentへ押し込みすぎない
6. 複数Data Componentの協調はUseCaseで扱える
7. 依存先の変更をFeature自身の変更理由と混同しない
8. Entry固有のUseCaseを許容する
9. Feature自身も分割・統合可能な仮説として扱う
10. 他概念からの利用にはFeatureの公開面を使う

次章では、Featureから利用される業務概念側、すなわちData ComponentとRepositoryを具体化する。

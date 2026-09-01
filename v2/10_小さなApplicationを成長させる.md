# 第10章　小さなApplicationを成長させる

## この章の目的

一つの小さなApplicationが成長する過程を通して、観測した事実から概念候補を作り、配置、公開面、依存、Transactionを更新するまでの導出を示す。

この章の構造はReference Architectureではない。別のApplicationでも同じフォルダになることを主張せず、どの事実から何を判断したかを事例として示す。

## 1. Job登録だけから始める

最初の要求は、Jobを登録することだけとする。

```text
app.py
```

```python
jobs: list[dict] = []


def create_job(name: str) -> dict:
    job = {
        "id": len(jobs) + 1,
        "name": name,
        "status": "created",
    }
    jobs.append(job)
    return job
```

### 観測

- 利用目的はJob登録一つだけである
- 外部入力、永続化、状態規則の差はまだ小さい

### 暫定判断

一つのModuleへ置く。将来必要になりそうなRepositoryやDTOを先に作らない。

### 再検討する条件

異なる利用目的、外部経路、状態規則、永続化責任を説明できるようになったとき。

## 2. WebとDatabaseを追加する

Web RequestからJobを登録し、Databaseへ保存する。

```python
def create_job_handler(request):
    name = request.form["name"]
    connection = connect_database()
    connection.execute(...)
    connection.commit()
    return render_template(...)
```

### 観測

- HTTP RequestとResponseはWebに由来する
- Job登録はWeb以外からも成立し得る利用目的である
- Database利用は増えたが、Jobについて複数の永続化操作はまだない

### 候補

1. すべて`app.py`へ残す
2. WebだけをEntryとして分ける
3. Web、Feature、Repository、Domainを一度に作る

### 暫定判断

Webという外部経路だけを分ける。

```text
entry/
    web.py
app.py
```

Job登録が一つの処理として十分に読める間は、FeatureやRepositoryの細分化を保留する。

## 3. 状態変更をFeatureとして並べる

Change Job StatusとComplete Jobが追加された。

処理列を先に置く。

```python
def execute(...):
    # 対象Jobを確認する

    # Jobの状態を変更する

    # 変更を確定する
```

### 観測

- Create、Change、Completeは異なる利用目的として説明できる
- 各処理には独自の入力と成功条件がある
- まだ状態規則が同じ概念へ集まるかは確定していない

### 暫定判断

利用目的をFeatureとして並べる。

```text
feature/
    create_job.py
    change_job_status.py
    complete_job.py

entry/
    web.py
```

Feature内部を`command.py`、`handler.py`、`service.py`等へ一式分割しない。一つのModuleで始める。

## 4. Job Data Componentを作る

Featureが増え、次の規則が現れた。

- Completed Jobは変更できない
- Running Jobだけを完了できる
- Cancelled Jobは再開できない

### 観測

- 複数Featureが同じJob状態を扱っている
- 状態遷移規則はFeature固有の入力条件ではなく、Jobに共通している
- Jobには識別子とライフサイクルがある

### 候補

1. 規則を各Featureへ残す
2. `shared/job_rules.py`へ置く
3. `data_component/job.py`を作る

### 暫定判断

Jobの状態、同一性、規則を中心に説明できるため、Data Componentを作る。

```text
data_component/
    job.py

feature/
    create_job.py
    change_job_status.py
    complete_job.py
```

FeatureはJobの公開操作を利用する。Feature固有の入力条件はFeatureへ残す。

## 5. Repositoryが具体化する

Jobの取得と保存が複数の公開操作で必要になった。

### 観測

- SQLが複数Featureへ散り始めた
- 取得結果からJobを復元する処理が繰り返される
- Jobの永続化という内部役割を説明できる

### 暫定判断

`job.py`を同名Packageへ成長させる。

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
        service.py
```

`__init__.py`から公開操作を示す。

```python
from .service import complete_job

__all__ = ["complete_job"]
```

Featureは内部Repositoryを直接利用せず、次の公開面を使う。

```python
from data_component.job import complete_job
```

Repositoryはcommitしない。Transactionは利用目的を知るFeature側で確定する。

## 6. Scheduleを分割する

Reschedule Jobが増え、最初はScheduleをJob内部へ置いた。

```text
data_component/
    job/
        schedule.py
```

その後、次の事実が増えた。

- Job完了後もScheduleを変更できる
- Scheduleだけを作り直せる
- Schedule固有の状態とRepositoryがある
- Jobとは異なるFeatureからScheduleが利用される

### 候補

1. Job内部へ残す
2. Job内部でSchedule Packageを作る
3. Jobと兄弟のData Componentへ分ける

### 暫定判断

JobがなくてもScheduleの状態とライフサイクルを説明できるため、兄弟へ分ける。

```text
data_component/
    job/
    schedule/
```

Reschedule Jobは両方の公開操作を利用する。

```python
def execute(command, *, dependencies):
    dependencies.ensure_reschedulable(
        command.job_id,
        dependencies.connection,
    )
    dependencies.change_period(
        command.schedule_id,
        command.period,
        dependencies.connection,
    )
    dependencies.connection.commit()
```

Data Componentを分けても、同じConnectionによって一つのTransactionにできる。

## 7. 読み取りFeatureを追加する

Job一覧表示が必要になった。

### 観測

- 一覧表示自体が利用目的である
- 必要なのはJob ID、名称、状態、予定日だけである
- 状態変更用の完全なJob Objectを復元する必要はない

### 暫定判断

読み取りだけのFeatureを作り、必要なRead Modelを直接返す。

```text
feature/
    list_jobs.py
```

```python
@dataclass(frozen=True, slots=True)
class JobListItem:
    job_id: int
    name: str
    status: str
```

Webから使われるという理由だけで、Database Queryを`entry/web/`へ置かない。配置理由はList Jobsという利用目的である。

## 8. 最終構造は暫定結果である

この時点では、例えば次の構造になる。

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
        service.py
    schedule/
        __init__.py
        domain.py
        repository.py
        service.py

feature/
    create_job.py
    complete_job.py
    reschedule_job.py
    list_jobs.py

entry/
    web/
        routes.py
    cli.py

app.py
```

この構造は、ここまでの観測から説明できる暫定結果である。

- Notificationに独立した利用目的が見えればFeatureを追加できる
- 外部サービス固有責任が増えれば`external_service/`を追加できる
- JobとScheduleの意味が再び一体だと分かれば統合を検討できる
- Feature内部にQueryや補助役割が増えればPackage化できる

## 9. 既存Applicationから移行する

既存構造が役割別に分かれている場合も、全体を一度に移す必要はない。

```text
domain/
service/
repository/
web/
```

Complete Jobを変更する機会に、その利用目的とJob規則を取り出せる。

```text
feature/
    complete_job.py

data_component/
    job.py
```

移行中は新旧構造が共存する。互換用公開面、Testの移行、import path、二重化を解消する終了条件を決める。

`service/`を`feature/`へ名前変更するだけでは、意味単位への再構成にならない。現在変更する具体的な意味を手掛かりに、責任と配置を一緒に見直す。

## 10. この事例が示すことと示さないこと

この事例が示すのは、次の判断過程である。

- 異なる利用目的が見えてからFeatureを作る
- 同じ状態と規則が複数Featureへ現れてからData Componentを作る
- 内部役割が増えてからModuleをPackageへ成長させる
- 独立したライフサイクルが見えてからData Componentを分割する
- Queryを読み取りの利用目的へ置く
- 配置変更に合わせて公開面、依存、Transactionを更新する

この事例だけでは、すべてのApplicationにJobやScheduleが存在することも、同じ段階で同じ構造になることも示さない。

重要なのは最終treeではなく、観測事実から配置へ進む途中である。

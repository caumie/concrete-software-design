# 第5章　Data ComponentとRepository

## 1. Data Componentは「意味あるデータ」を中心にしたまとまりである

具象設計論では、システムが扱う意味あるデータを中心に、状態・規則・永続化・そのデータを扱う公開操作をまとめる単位を **Data Component** と呼ぶ。

例えばJobというデータ概念が見えたなら、最初は、

```text
data_component/
    job.py
```

から始められる。

```python
# data_component/job.py

class Job:
    def __init__(
        self,
        job_id: int,
        status: str,
    ):
        self.id = job_id
        self.status = status

    def complete(self) -> None:
        if self.status != "running":
            raise ValueError(
                "only running job can be completed"
            )

        self.status = "completed"
```

Data Componentという名前の「Data」は、Database RowやDTOだけを意味しない。

ここでいうDataは、

> システムが状態として保持し、参照し、変更する意味ある対象

を指す。

したがってData Componentは、単なるData Access Layerではない。必要なら、

- 状態
- 業務規則
- Repository
- 公開操作
- そのデータに固有の補助処理

を内部に持てる。

Data Componentはコード配置上の意味単位であり、Transaction境界は別に判断する。

Repositoryや複数の公開処理など、Job内部に分けて扱う役割が増えたときに`data_component/job/`へPackage化すればよい。

## 2. 業務ロジックがあるだけでData Componentへ分離しない

業務上の判断が一つ現れたからといって、それをFeatureからData Componentへ移す義務はない。

最初のFeatureの内部に、

```python
if status == "completed":
    ...
```

のような判断があってもよい。

具体例が増え、

- 同じデータの状態を扱う
- 複数Featureで同じ規則が必要になる
- 独立したライフサイクルや同一性が見える
- Featureの利用目的とは別に、そのデータ自身の意味として説明できる

と分かったときに、Data Componentとして分離する意味が生まれる。

具象設計論は「Domain LogicはDomain層へ分ける」という規則を置かない。

**データ中心の概念として独立させる理由が具体化したときにData Componentを作る。**

## 3. EntityとValue Objectも必要になったときに分ける

JobIdに独立した意味が必要になったとする。

```python
@dataclass(frozen=True, slots=True)
class JobId:
    value: int
```

この型によって、

- 他のIDとの取り違えを防ぐ
- JobIdとしてのValidationを持つ
- 型として意味を示す

といった価値が得られるなら作る。

しかし、単に`int`を一枚包むことしかしていない段階で、すべてのIDをValue Object化する必要はない。

Entity、Value Object、Domain ServiceといったDDDの役割も、具体的な必要性から導入する。

---

## 4. Repositoryが現れたらData ComponentをPackageへ成長させられる

永続化が必要になり、Job本体とRepositoryを別の役割として扱う意味が生じたなら、`job.py`をPackageへ成長させる。

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
```

```python
# data_component/job/repository.py

from .domain import Job


def get(
    connection,
    job_id: int,
) -> Job:
    row = connection.execute(
        """
        SELECT id, status
        FROM jobs
        WHERE id = ?
        """,
        (job_id,),
    ).fetchone()

    if row is None:
        raise LookupError(job_id)

    return Job(
        job_id=row["id"],
        status=row["status"],
    )


def save(
    connection,
    job: Job,
) -> None:
    connection.execute(
        """
        UPDATE jobs
        SET status = ?
        WHERE id = ?
        """,
        (job.status, job.id),
    )
```

このRepositoryはJobについての永続化責任を持つため、Jobの配下に置く。

Repositoryが一つ増えたから新しい技術層を作るのではない。

**Jobという概念の内部に、DomainとRepositoryという二つの役割が具体化したためPackage化する。**

`job/__init__.py`で従来の公開面を保てば、利用側は`data_component.job`という同じ概念名を使い続けられる。

## 5. RepositoryはData Componentに関する依存として扱える

Jobを取得・保存する処理をData Componentの公開機能が利用する場合、Repositoryはそのモジュールにとって外部依存になる。

例えば、

```text
data_component/
    job/
        domain.py
        repository.py
        service.py
```

とする。

`service.py`では、

```python
def complete_job(
    job_id: int,
    connection,
):
    job = repository.get(
        connection,
        job_id,
    )

    job.complete()

    repository.save(
        connection,
        job,
    )
```

のような処理を持てる。

ここでRepositoryの依存をどのように明示し、標準構成とテスト差し替えをどう扱うかは、第6章・第7章で詳しく扱う。

重要なのは、Feature側がRepositoryの詳細を知らなくても、Job Data Componentの公開機能を利用できる構造を作れることである。

---

## 6. Data Componentの公開機能を作る

Jobを外部から利用するために、

```python
# data_component/job/service.py

def complete_job(
    job_id: int,
    connection,
):
    job = repository.get(
        connection,
        job_id,
    )

    job.complete()

    repository.save(
        connection,
        job,
    )
```

とする。

そして、

```python
# data_component/job/__init__.py

from .domain import Job
from .service import complete_job

__all__ = [
    "Job",
    "complete_job",
]
```

と公開する。

Featureからは、

```python
from data_component.job import complete_job
```

を利用する。

Job内部でどのRepositoryを使うかまでFeatureが知る必要はない。

---

## 7. Repositoryは対象概念の近くに置く

Jobの永続化をJobという意味の一部として扱うなら、Repositoryもその近くへ置く。

```text
data_component/
    job/
        repository.py
```

という配置なら、Jobの変更を見るときに永続化処理も同じ概念配下で確認できる。

> 変更しようとしている意味の近くへ必要なコードを置くこと

を配置へ反映した形である。永続化方式が独立した概念として説明できる場合は、その意味に応じて別の配置を選べる。

---

## 8. Repositoryを巨大な共通データアクセス層にしない

望ましくない例として、

```python
class Repository:
    def get_job(*args, **kwargs):
        ...

    def save_job(*args, **kwargs):
        ...

    def get_schedule(*args, **kwargs):
        ...

    def save_schedule(*args, **kwargs):
        ...

    def get_user(*args, **kwargs):
        ...

    def save_user(*args, **kwargs):
        ...
```

のような巨大なRepositoryがある。

これは複数概念の永続化を一つの技術的役割へまとめている。

結果として、どの変更理由でこのRepositoryが変わるのか分からなくなる。

Jobについての永続化ならJobへ置く。

Scheduleについての永続化ならScheduleへ置く。

```text
data_component/
    job/
        repository.py

    schedule/
        repository.py
```

と分ける。

同じDatabaseを使っていることは、同じ概念である理由にはならない。

---

## 9. Data Componentを分けてもUseCaseで協調できる

JobとScheduleを別Data Componentへ分ける。

```text
data_component/
    job/
    schedule/
```

Featureからは、それぞれの公開機能を利用する。

```python
from data_component.job import ensure_reschedulable
from data_component.schedule import change_period


def execute(
    job_id,
    schedule_id,
    period,
    connection,
):
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

Data Componentを分けることは、両方を一つのUseCaseから利用できなくすることではない。

意味の境界と実行の境界は別に考える。

---

## 10. Data Component境界とTransaction境界を同一視しない

Data Componentは、意味あるデータを中心にコードをまとめる単位である。

Transactionは、

> どこまでを一緒に成功させ、どこまでを一緒に失敗させるか

という実行上の境界である。

したがってJobとScheduleが別Data Componentであっても、一つのUseCaseで同じConnectionを使って一Transactionとして扱える。

```python
job_operation(..., connection)
schedule_operation(..., connection)

connection.commit()
```

- Jobの状態・規則・永続化はJob Data Component
- Scheduleの状態・規則・永続化はSchedule Data Component
- 両者をどう組み合わせるかはFeature / UseCase
- どこでcommit / rollbackするかはTransaction設計

として別々に判断する。

## 11. Data Componentも大きくなりすぎる

Job Data Componentが、

```text
data_component/
    job/
        domain.py
        repository.py
        schedule.py
        assignment.py
        revision.py
        delivery.py
        calculation.py
```

と増えたとする。

この状態だけで分割が必要とは限らない。

しかし、具体例が増えたことで新しい問いを立てられる。

ScheduleはJobと同じライフサイクルか。RevisionはJob本体の状態なのか、履歴なのか。Assignmentは独立した変更理由を持たないか。

Data Componentという名前を守るために、すべてを内部へ残す必要はない。

Data Component自身も、具体化によって分割・統合される仮説である。

---

## 12. Repositoryも概念発見の観測点になる

Repositoryが巨大化して、

```python
def get_job(*args, **kwargs): ...
def get_schedule(*args, **kwargs): ...
def find_available_members(*args, **kwargs): ...
def load_revision_history(*args, **kwargs): ...
def search_jobs_for_dashboard(*args, **kwargs): ...
```

と増えたとする。

これは単なるファイルサイズの問題ではない。

Jobの永続化、Scheduleの永続化、Dashboard用検索など、異なる意味が混ざっている可能性がある。

Repositoryの増大は、Data Component境界やQuery責任を見直す観測点になる。

---

## 13. Data ComponentとRepositoryの判断原則

この章の内容をまとめる。

1. Data Componentは具体的な業務概念として必要になったとき作る
2. DDDの役割を一式先に作らない
3. Repositoryは対象となる概念の近くへ置く
4. Repositoryを別概念として切り出す場合は、その独立した意味を説明する
5. 巨大な共通Repositoryを作らない
6. Data Componentの公開面からFeatureが利用できる
7. Data Componentを分けてもUseCaseで協調できる
8. Data Component境界とTransaction境界を同一視しない
9. Data ComponentもRepositoryも具体化によって再評価する

次章では、このData ComponentやFeature内部で使われる外部要素を、どのように「依存」として明示するかを扱う。

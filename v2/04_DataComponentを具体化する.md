# 第4章　Data Componentを具体化する

## この章の目的

複数のFeatureに現れた状態、規則、同一性、永続化から、意味あるデータを中心とするData Componentを具体化する。Data ComponentとDomain、Repository、Transactionの境界を混同しない。

## 1. Data Componentは意味あるデータを中心にする

Data Componentは、システムが状態として保持し、参照し、変更する意味ある対象を中心にしたコード配置上の単位である。

Jobという概念なら、最初は一つのModuleで始められる。

```text
data_component/
    job.py
```

```python
class Job:
    def __init__(self, job_id: int, status: str):
        self.id = job_id
        self.status = status

    def complete(self) -> None:
        if self.status != "running":
            raise ValueError("only running job can be completed")
        self.status = "completed"
```

ここでいうDataは、Database RowやDTOだけを意味しない。状態、規則、同一性、ライフサイクルを持つ対象を含む。

## 2. 一つの規則だけで分離しない

Featureに業務上の条件が一つ現れたからといって、直ちにData Componentへ移す必要はない。

次の事実が増えたとき、独立したデータ概念を検討できる。

- 複数Featureが同じ状態を扱う
- 同じ状態遷移規則が繰り返し現れる
- Featureの利用目的とは別の同一性を持つ
- 独立した作成、変更、終了のライフサイクルがある
- そのデータ自身について公開したい操作がある

これらは採用条件の一覧ではなく、何を一つの概念として扱うかを検討する材料である。

### 例

Create Job、Complete Job、Cancel Jobの各Featureに、Job状態を読む処理と遷移規則が現れたとする。

- 各Featureへ残せば、利用目的ごとに局所化できる
- `shared/job_rules.py`へ置けば、広い共有規則として扱える
- `data_component/job.py`へ置けば、Jobの状態と同一性を中心に説明できる

規則がJob状態のどの利用にも共通し、Jobとして独立したライフサイクルを持つと判断したなら、Data Componentを作る理由になる。

## 3. Domain発見とData Componentを同一視しない

Data Componentを作っても、それだけで適切なDomain Modelが発見されるわけではない。

何が業務上重要か、どの言葉が同じ意味か、どの境界で規則が変わるかは、業務分析や利用者との対話から得る。

Data Componentは、その時点で得た理解をコードへ置く単位である。理解が変われば、Data Componentも分割・統合する。

DDDのEntity、Value Object、Aggregate、Domain Serviceなどは、その意味が具体化したときに利用できる。

例えばJobIdに独立した検証や型としての意味があるなら、Value Objectとして表現できる。

```python
@dataclass(frozen=True, slots=True)
class JobId:
    value: int
```

単に`int`を包むだけで価値がない段階では、作らなくてもよい。

## 4. 内部役割が増えたらPackageへ成長させる

永続化と公開操作が増えたら、`job.py`を同名Packageへ成長させられる。

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
        service.py
```

- `domain.py`はJobの状態と規則
- `repository.py`はJobの永続化
- `service.py`はJobを取得し、規則を適用し、保存する公開操作

という内部役割を持てる。

最初からこの一式を作るのではない。複数の役割を分ける理由が具体化したためPackageへ進む。

## 5. Repositoryは対象概念の近くへ置く

Jobを取得・保存するRepositoryなら、Jobの配下へ置ける。

```python
# data_component/job/repository.py

class DefaultJobRepository:
    def get(self, connection, job_id: int) -> Job:
        ...

    def save(self, connection, job: Job) -> None:
        ...
```

RepositoryをJobの配下へ置くことと、Jobの公開操作がRepository実装を直接参照することは別である。

RepositoryはJob内部のServiceが直接利用する依存として、Service側の`Dependencies`へ明示できる。通常利用するRepository実装は、Service近傍の`default()`で選ぶ。FeatureへRepositoryを渡させず、Job内部で標準構成を解決する。

Repositoryという技術役割だけを理由に、すべてのRepositoryを一つのトップレベルへ集めない。

```python
class Repository:
    def get_job(...): ...
    def save_job(...): ...
    def get_schedule(...): ...
    def save_schedule(...): ...
```

のような共通Repositoryは、複数の意味を一つの変更単位へまとめやすい。

一方、永続化基盤自体が独立した責任、公開面、変更理由を持つなら、別概念として配置する選択もできる。Repositoryは必ずData Component内部、という禁止規則ではない。

## 6. Data Componentの公開操作を作る

FeatureがJob内部のRepositoryやDomain Objectを個別に操作する代わりに、Jobの公開操作を利用できる。

```python
# data_component/job/service.py

from dataclasses import dataclass
from typing import Protocol, Self

from .domain import Job


class JobRepository(Protocol):
    def get(self, connection, job_id: int) -> Job:
        ...

    def save(self, connection, job: Job) -> None:
        ...


@dataclass(frozen=True, slots=True)
class Dependencies:
    repository: JobRepository

    @classmethod
    def default(cls) -> Self:
        from .repository import DefaultJobRepository

        return cls(
            repository=DefaultJobRepository(),
        )


def complete_job(
    job_id: int,
    connection,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    job = dependencies.repository.get(connection, job_id)
    job.complete()
    dependencies.repository.save(connection, job)
```

`complete_job`の処理本体は、Repository実装ではなく、自分が直接利用するJob Repositoryの契約だけを知る。`Dependencies.default()`は、その標準実装を選ぶ場所である。

ServiceのModule Testでは、`repository`を差し替える。Featureはこの内部依存を構成せず、公開された`complete_job`を利用する。

```python
# data_component/job/__init__.py

from .service import complete_job

__all__ = ["complete_job"]
```

Featureは次の入口を利用する。

```python
from data_component.job import complete_job
```

この公開操作によって、Jobをどう取得し、どの内部役割を使うかをJob側で変更できる。
RepositoryはJob Serviceの直接依存として同じModuleで明示し、標準構成もその近傍で解決する。FeatureへRepositoryを受け渡させない。

## 7. Data Component同士はUseCaseで協調できる

JobとScheduleを分けても、一つの利用目的から両方を利用できる。

```python
from data_component.job import ensure_reschedulable
from data_component.schedule import change_period


def execute(command, connection):
    ensure_reschedulable(command.job_id, connection)
    change_period(command.schedule_id, command.period, connection)
    connection.commit()
```

JobがSchedule Repositoryを直接知る必要はない。どの概念をどの順序で利用するかはReschedule Job Featureが扱う。

## 8. Data Component境界とTransaction境界を分ける

Data Componentは、意味あるデータを中心にコードを配置する単位である。

Transactionは、どこまでを一緒に成功させ、どこまでを一緒に失敗させるかという実行上の境界である。

別Data Componentでも同じTransactionで変更できる。同じData Componentでも、異なる利用目的が別Transactionになることがある。

一緒にcommitするという事実だけで、一つのData Componentにまとめない。

## 9. 境界を再評価する

Job配下が次のように増えたとする。

```text
data_component/
    job/
        domain.py
        repository.py
        schedule.py
        assignment.py
        revision.py
```

ファイル数だけでは分割しない。しかし、ScheduleがJob完了後も変更される、Revisionだけ独立して追加・削除される、Assignmentだけ別の利用目的から頻繁に変更される、といった事実は新しい境界候補になる。

Data Componentは完成形ではなく、その時点の概念理解を置いた仮説である。

## 10. この章の結論

Data Componentは、意味あるデータの状態、規則、同一性、永続化、公開操作を近くへ置くための単位である。

業務ロジックが一つあるだけで作るものではなく、Domain発見を自動化するものでもない。複数の具体例から独立したデータ概念が見えたときに作り、理解の更新に応じて分割・統合する。Transaction境界は別に判断する。

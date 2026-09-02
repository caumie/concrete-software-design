# 第5章　Data Componentを具体化して構造を成長させる

## この章の目的

前二章で増やしたFeatureから、同じ状態、規則、同一性、永続化、ライフサイクルを観測し、Job Data Componentを具体化する。さらに、内部役割が増えたときのPackage化、Repositoryと公開操作の配置、別Data Componentとの協調、境界の再評価までを、同じApplicationの成長として示す。

Data Componentは、業務上の名詞をすべてフォルダへ変換する規則ではない。適切なDomainを自動的に発見する仕組みでもない。その時点で意味あるデータとして説明できるものを、状態と操作の近くへ置くためのコード配置上の単位である。

## 1. 意味あるデータを中心にする

Data Componentは、システムが状態として保持し、参照し、変更する意味ある対象を中心にする。

Jobなら、最初は一つのModuleで始められる。

```text
data_component/
    job.py
```

```python
# data_component/job.py

from dataclasses import dataclass


@dataclass
class Job:
    job_id: int
    status: str

    def complete(self) -> None:
        if self.status != "running":
            raise ValueError("only running job can be completed")
        self.status = "completed"

    def cancel(self) -> None:
        if self.status == "completed":
            raise ValueError("completed job cannot be cancelled")
        self.status = "cancelled"
```

ここでいうDataは、Database Rowや外部入出力のDTOだけを意味しない。

- 状態
- 同一性
- 状態遷移の規則
- 作成から終了までのライフサイクル
- 取得と保存
- 別概念へ公開する操作

を含めて、Jobという意味の近くへ置ける。

一方、Requestの一時的な入力、画面表示のためだけに組み立てた値、処理途中で一度だけ使う中間値まで、すべてData Componentにする必要はない。

## 2. 一つの規則だけで分離しない

Featureに業務上の条件が一つ現れたからといって、直ちにData Componentへ移す必要はない。

例えばComplete Jobにだけ、完了理由を必須にする条件がある。

```python
if not command.reason.strip():
    raise ValueError("completion reason is required")
```

この条件がComplete Jobという利用目的にだけ属するなら、Featureへ残せる。

別のData Componentを検討する材料になるのは、次のような事実である。

- 複数Featureが同じ状態を扱う
- 同じ状態遷移規則が繰り返し現れる
- Featureの利用目的とは別の同一性を持つ
- 独立した作成、変更、終了のライフサイクルがある
- そのデータ自身について公開したい操作がある
- 取得結果から同じ状態を復元し、同じ形で保存する処理が増える

これらは採用条件のChecklistではない。一項目を満たしたら自動的にフォルダを作るのではなく、何を一つの意味として扱うかを検討する材料にする。

### Jobの例

Create Job、Complete Job、Cancel Job、Change Job Statusの各Featureに、Job状態を読む処理と遷移規則が現れたとする。

候補は少なくとも三つある。

1. 各Featureへ規則を残す
2. `shared/job_rules.py`へ置く
3. `data_component/job.py`へ置く

各Featureへ残すと、利用目的ごとの変更は局所化できる。規則が似て見えても、実は利用目的ごとに異なるなら、この選択が正しい場合もある。

`shared/job_rules.py`へ置くと、複数箇所から利用できる。しかし「共有される」という利用範囲しか親概念を説明できず、Jobの状態と同一性が分離されやすい。

`data_component/job.py`へ置くと、規則をJobの状態、同一性、ライフサイクルと一緒に説明できる。

規則がJob状態のどの利用にも共通し、Jobとして独立したライフサイクルを持つと判断したため、本事例ではJob Data Componentを暫定採用する。

## 3. Domain発見とData Componentを同一視しない

Data Componentを作っても、それだけで適切なDomain Modelが発見されるわけではない。

何が業務上重要か、どの言葉が同じ意味か、どこで規則が変わるかは、利用者との対話、業務分析、規程、既存システム、運用経験などから得る。

Jobという名前を置いた後にも、次の誤りは起こり得る。

- 異なる種類のJobを一つにまとめている
- 実際にはJobと一緒に変わらないScheduleを内包している
- Feature固有の入力条件をJob共通の規則としている
- Database Tableの単位を、そのまま業務概念だとみなしている
- `Job`という一般語が、関係者ごとに違う意味で使われている

Data Componentは、その時点で得た理解をコードへ置く単位である。理解が変われば、分割、統合、移動、改名する。

DDDのEntity、Value Object、Aggregate、Domain Serviceなどは、具体化した意味を説明するのに役立つときに使える。

例えばJob IDに独立した検証や型としての区別が必要なら、Value Objectとして表現できる。

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class JobId:
    value: int
```

単に`int`を包むだけで、誤用防止、検証、振る舞い、型上の区別のいずれも得られない段階では、作らなくてもよい。Data Componentを作ったことを理由に、関連する設計用語を一式実装しない。

## 4. 内部役割が増えてからPackageへ成長させる

最初のJob Data Componentは、一つのModuleだった。

```text
data_component/
    job.py
```

その後、次の事実が増えたとする。

- Jobの状態と遷移規則が増えた
- Database RowからJobを復元する処理が複数の操作で必要になった
- Jobを取得し、規則を適用し、保存する公開操作が増えた
- 別概念から通常利用させる入口を明示したい

この時点で、`job.py`を同名Packageへ成長させられる。

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
        service.py
```

この事例では、次の内部役割を区別する。

| ファイル | 置く内容 | 主な変更理由 |
| --- | --- | --- |
| `domain.py` | Jobの状態、同一性、状態規則 | Jobの意味や規則が変わる |
| `repository.py` | Rowとの変換、取得、保存 | Schemaや永続化方法が変わる |
| `service.py` | Jobを取得し、規則を適用し、保存する公開操作 | Jobの通常利用方法が変わる |
| `__init__.py` | 別概念から使う公開面 | 公開契約が変わる |

これは、Data Componentなら必ず四ファイル必要だというTemplateではない。内部役割を別々に説明し、変更・Testする理由が現れたため分けている。

Jobが一つのClassと数個の関数で十分なら、`job.py`へ戻してもよい。

## 5. Repositoryは対象概念の近くへ置く

Jobを取得・保存し、Database表現からJob状態を復元するRepositoryなら、Jobの配下へ置ける。

```python
# data_component/job/repository.py

from .domain import Job


class DefaultJobRepository:
    def get(self, connection, job_id: int) -> Job:
        row = connection.execute(
            "SELECT id, status FROM job WHERE id = ?",
            (job_id,),
        ).fetchone()

        if row is None:
            raise LookupError(f"job not found: {job_id}")

        return Job(
            job_id=row["id"],
            status=row["status"],
        )

    def save(self, connection, job: Job) -> None:
        connection.execute(
            "UPDATE job SET status = ? WHERE id = ?",
            (job.status, job.job_id),
        )
```

RepositoryをJobの配下へ置くことと、Job Object自身がRepositoryを呼ぶことは別である。物理的な配下は親となる意味を表し、実行時の利用関係はimportと呼び出しで表す。

Repositoryという技術役割だけを理由に、すべてのRepositoryを一つへまとめない。

```python
class Repository:
    def get_job(...): ...
    def save_job(...): ...
    def get_schedule(...): ...
    def save_schedule(...): ...
```

この形は、JobとScheduleという別の意味を、一つの巨大な変更単位へまとめやすい。

一方、Application全体の保存境界や永続化基盤自体が、独立した責任、公開面、変更理由を持つなら、Application直下の`repository.py`や別の具体名を持つ親概念へ置く選択もできる。Repositoryは必ずData Component内部、という禁止規則ではない。

## 6. Data Componentの公開操作を作る

FeatureがJob内部のRepositoryとDomain Objectを個別に操作すると、Feature側がJobの取得、復元、状態規則、保存の順序を知る。

```python
# feature/complete_job.py

job = job_repository.get(connection, command.job_id)
job.complete()
job_repository.save(connection, job)
```

この手順をJobの通常利用として公開したいなら、Job Data Componentへ公開操作を作れる。

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

        return cls(repository=DefaultJobRepository())


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

`complete_job`は、自分が直接利用するJob Repositoryの契約だけを知る。Featureは、Job内部でどのRepository実装を使い、どの順序で取得・保存するかを知らなくてよい。

通常利用するRepository実装は、`service.py`に置いたModule localな`Dependencies.default()`で選べる。外側のCompositionから同じ`Dependencies`を明示的に渡す形も取れる。

```python
dependencies = Dependencies(repository=FakeJobRepository(...))

complete_job(
    job_id=10,
    connection=fake_connection,
    dependencies=dependencies,
)
```

ここで確保しているのは、Job Serviceの処理本体がRepository実装と分かれ、Module Testで直接依存を差し替えられることである。

Poolや長寿命Clientの生成・破棄まで、この`default()`が引き受けるとは限らない。標準構成の選択、実行時scope、Resource lifecycleは第7章と第14章で分けて扱う。

## 7. 公開面から通常利用する入口を示す

Job Packageの`__init__.py`から、別概念が通常利用するものを公開する。

```python
# data_component/job/__init__.py

from .service import complete_job

__all__ = ["complete_job"]
```

Featureは公開面を利用する。

```python
from data_component.job import complete_job
```

Pythonでは、次のように内部Moduleを直接importすることもできる。

```python
from data_component.job.repository import DefaultJobRepository
```

公開面の目的は、物理的に内部参照を不可能にすることではない。別概念から通常どこを入口にするかを示し、深いimportが増えたときに境界を再検討できるようにすることである。

同じJob Package内部では、`from .repository import DefaultJobRepository`のような直接参照を過剰に迂回させる必要はない。概念内部と概念外部では、必要な安定性が異なる。

## 8. Data Component同士をFeatureで協調させる

Reschedule Jobが追加され、JobとScheduleの両方を変更するとする。

JobとScheduleが別Data Componentでも、一つの利用目的から両方の公開操作を利用できる。

```python
# feature/reschedule_job.py

from data_component.job import ensure_reschedulable
from data_component.schedule import change_period


def execute(command, connection) -> None:
    ensure_reschedulable(
        command.job_id,
        connection,
    )
    change_period(
        command.schedule_id,
        command.period,
        connection,
    )
    connection.commit()
```

JobがSchedule Repositoryを直接知る必要はない。Scheduleも、どのFeatureから利用されたかを知る必要はない。

Reschedule Job Featureは、次を扱う。

- JobとScheduleをどの順序で利用するか
- 両方の変更が揃うことを何とみなすか
- どこでcommitするか
- 失敗を利用者へどう返すか

Data Componentの公開操作は、Featureから利用される部品というだけではない。各概念の状態と規則を、その概念の通常利用として公開する。

## 9. Data Component境界とTransaction境界を分ける

Data Componentは、意味あるデータを中心にコードを配置する境界である。

Transactionは、どこまでを一緒に成功させ、どこまでを一緒に失敗させるかという実行上の境界である。

JobとScheduleが別Data Componentでも、同じConnectionを渡せば一つのTransactionで変更できる。

反対に、同じJob Data Componentを使うCreate JobとComplete Jobが、別々のTransactionになることもある。

| 観点 | この事例での境界 |
| --- | --- |
| 意味あるデータ | JobとScheduleを別Data Componentにする |
| 利用目的の成功 | Reschedule Jobが両方を一緒にcommitする |

一緒にcommitするという事実だけで、一つのData Componentへまとめない。別々に保存できるという事実だけで、意味まで分けない。

Transaction境界は利用目的から決め、第8章で詳しく扱う。

## 10. ScheduleをJobから分割する

Reschedule Jobを最初に実装したとき、ScheduleをJob内部へ置いたとする。

```text
data_component/
    job/
        domain.py
        repository.py
        schedule.py
        service.py
```

この配置は、ScheduleがJobの一部としてしか意味を持たない段階では成立する。

その後、次の事実が増えた。

- Job完了後もScheduleを変更できる
- Scheduleだけを作り直せる
- Schedule固有の状態とRepositoryがある
- Jobとは異なるFeatureからScheduleが利用される
- Scheduleの変更頻度とライフサイクルがJobと異なる

### 配置候補

1. Job内部へ残す
2. Job内部でSchedule Packageへ深掘りする
3. Jobと兄弟のData Componentへ分ける

### 候補の差

Job内部へ残すと、Jobと常に一緒に理解・変更する関係を表せる。

Job内部のSchedule Packageにすると、内部役割は分けられるが、Scheduleは引き続きJobという親概念の下にある。

兄弟Data Componentへ分けると、Scheduleを独立した状態、同一性、ライフサイクルとして公開できる。その代わり、import path、公開面、Dependencies、Test、移行中の互換性を更新する費用が発生する。

### 暫定判断

JobがなくてもScheduleの状態とライフサイクルを説明できるため、兄弟のData Componentへ分ける。

```text
data_component/
    job/
    schedule/
```

この判断は、ファイル数が増えたからではない。親であるJobがなくても、Scheduleが同じ意味で成立すると判断したためである。

分割後もReschedule Jobは、両方の公開操作へ同じConnectionを渡し、一つのTransactionとして協調できる。

## 11. 成長後の構造も暫定結果である

ここまでの観測から、Applicationは例えば次の構造になる。

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

entry/
    web/
        routes.py
    cli.py

app.py
```

このtreeは、最初に目指した完成形ではない。

1. `app.py`一つでJob登録を始めた
2. Web固有処理が現れ、Entryを分けた
3. 異なる利用目的が現れ、Featureを並べた
4. 複数FeatureにJob状態の規則が現れ、Job Data Componentを作った
5. 永続化と公開操作が増え、JobをPackageへ深掘りした
6. Scheduleの独立したライフサイクルが観測され、兄弟へ分割した

という判断の履歴である。

この後にも構造は変わり得る。

- Notificationに独立した利用目的が見えればFeatureを追加できる
- 外部サービス固有の通信と失敗制御が増えれば`external_service/`を追加できる
- JobとScheduleが常に同じ状態とライフサイクルで変わると分かれば再統合を検討できる
- Feature内部に独立した役割が増えれば同名Packageへ成長させられる
- Web固有だと思っていた処理が別Entryでも成立するならFeatureへ移せる
- 名前が実態を説明しなくなれば改名できる

構造は、その時点の概念理解を置いた暫定結果である。

## 12. 既存Applicationから段階的に移行する

既存Applicationが役割別の構造を持つ場合も、全体を一度に移す必要はない。

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

移行中は新旧構造が共存する。共存を許容するだけでなく、終了条件を決める。

- 旧import pathをどこまで互換公開するか
- 既存TestのPatch先をいつ移すか
- 同じ規則の二重実装をいつ解消するか
- 新旧の依存解決をどこで接続するか
- 移行対象外のModuleをどう識別するか
- 何をもって旧フォルダを削除できるか

`service/`を`feature/`へ名前変更するだけでは、意味単位への再構成にならない。現在変更する具体的な利用目的と状態を手掛かりに、責任、公開面、依存を一緒に見直す。

## 13. 変更可能性と変更費用を分ける

ModuleをPackageへ変えること、JobからScheduleを分けること、既存構造から段階移行することは可能である。しかし、可能であることは、痛みなく変更できることを意味しない。

Job Data Componentの分割では、少なくとも次が変わり得る。

- ファイルとフォルダ
- import pathと公開面
- RepositoryとDependenciesの定義場所
- Featureが利用する関数
- Transactionへ渡すConnection
- Test Fixture、Fake、Patch先
- Migrationや互換用Code
- 開発者が使う名称と文書

初期構造を小さくする目的は、この費用をゼロにすることではない。まだ観測していない境界を大量に固定せず、変更対象と影響範囲を狭く保つことである。

構造変更を行うときは、観測事実、候補、暫定判断に加えて、次も残す。

- どの利用と変更が局所化されるか
- どの公開契約が変わるか
- 段階移行できるか
- 互換層をいつ削除するか
- どの事実が増えたら再統合または再分割するか

根本的な救済策として変更費用を消すのではなく、支払う理由と範囲を説明できるようにする。

## 14. この事例が示すことと示さないこと

この事例が示すのは、次の判断過程である。

- 同じ状態と規則が複数Featureへ現れてからData Componentを作る
- 一つの規則だけで独立した概念を作らない
- Domain発見とコード配置を同一視しない
- 内部役割が増えてからModuleをPackageへ成長させる
- Repositoryを技術役割だけで集約せず、対象概念との関係から配置する
- 別概念からはData Componentの公開操作を利用する
- Data Component境界とTransaction境界を別に判断する
- 親概念なしでも成立する状態とライフサイクルが見えてから分割する
- 構造変更の費用と終了条件を明示する

この事例だけでは、すべてのApplicationにJobやScheduleが存在することも、一つのData Componentが一つのAggregateに対応することも、Repositoryが常に同じ場所へ置かれることも示さない。

重要なのは最終treeではなく、どの事実から概念候補を作り、何と比較し、どの影響を受け入れて配置したかという途中である。

## 15. この章の結論

Data Componentは、意味あるデータの状態、規則、同一性、永続化、公開操作を近くへ置くための単位である。

複数の具体例から独立したデータ概念が見えたときに一つのModuleとして始め、内部役割が増えたときに同名Packageへ成長させる。別概念からは公開面を利用し、直接依存は利用するModuleで明示する。Transaction境界は利用目的から別に決める。

Data Componentも完成形ではない。実装で増えた状態、変更理由、ライフサイクルを観測し、分割、統合、移動、改名を行う。その変更には費用が伴うため、理由、影響、移行、再検討条件を一緒に残す。

# 第14章　Pythonで標準構成とTestを実装する

## この章の目的

本書の依存原則をPythonで実装する一つの方法として、Module localな`Dependencies`、`default()`、`Protocol`、ModuleからPackageへの成長を示す。

Local Defaultで確定している性質と、長期運用でまだ検証が必要な性質を分ける。特に、Moduleの標準実装を選ぶことと、Connection Poolや長寿命Clientのlifecycleを構成することを同一視しない。

## 1. この章は言語固有の具体例である

言語を越えて残る原則は次である。

- Moduleが直接利用する依存を明示する
- 下位Moduleの依存を上位へ輸送しない
- 標準構成とTest差し替えの範囲を分ける
- 実装選択、Resource生成、lifecycle、受け渡し経路を区別する
- 通常構成が動くことと、処理本体が単体Testできることを別に確認する

`dataclass`、`Protocol`、`Dependencies.default()`は、これらをPythonで表現する具体案である。具象設計論の成立条件そのものではない。

## 2. ModuleからPackageへ同じ名前で成長する

Pythonでは、次のModuleを、同じimport pathを保ったPackageへ成長させられる。

```text
data_component/
    job.py
```

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
        service.py
```

`job/__init__.py`で通常の公開面を示す。

```python
from .service import complete_job

__all__ = ["complete_job"]
```

利用側は引き続き次の形を使える。

```python
from data_component.job import complete_job
```

Pythonでは深いimportを完全には禁止できない。`__init__.py`の公開面は、内部参照を不可能にする仕組みではなく、別概念からの通常入口を示す。

フォルダ化自体が前進なのではない。内部役割を分ける理由が具体化したためPackageへ進む。

## 3. Module localな`Dependencies`を定義する

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Dependencies:
    connection: Connection
    complete_job: CompleteJob
    send_notification: SendNotification
```

これはApplication全体のDependency Objectではない。このModuleの直接依存だけを表す。

`frozen=True`は構成後のfield差し替えを防ぎ、`slots=True`は意図しない属性追加を防ぐ。これらは本書の必須条件ではなく、Dependencyのまとまりを値として扱うための選択である。

依存が一つなら、普通の引数でよい。

```python
def calculate_deadline(start: date, calendar: Calendar) -> date:
    ...
```

依存を型へまとめるのは、複数の直接依存、Test差し替え、通常構成の入口を一つの単位として扱う意味があるときである。

## 4. 関数Dependencyの契約を表す

単純な形なら`Callable`で表現できる。

```python
CompleteJob = Callable[[int, Connection], None]
```

Keyword-only引数、overload、詳細な戻り値を型検査したい場合は`Protocol`を使える。

```python
class CompleteJob(Protocol):
    def __call__(
        self,
        job_id: int,
        connection: Connection,
        *,
        completed_at: datetime,
    ) -> None:
        ...
```

専用Classに包むことなく、関数の呼び出し能力をDependencyとして表せる。型を増やすこと自体を目的にしない。

## 5. `default()`で通常構成を選ぶ

Productionで通常使う実装がほぼ固定されているなら、Module近傍で表現できる。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    complete_job: CompleteJob
    send_notification: SendNotification

    @classmethod
    def default(cls) -> Self:
        from data_component.job import complete_job
        from external_service.mail import send_completion_notice

        return cls(
            complete_job=complete_job,
            send_notification=send_completion_notice,
        )
```

処理本体は、引数で受け取った直接依存だけを利用する。

```python
def execute(
    command: CompleteJobCommand,
    connection: Connection,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    dependencies.complete_job(
        command.job_id,
        connection,
        completed_at=command.completed_at,
    )
    dependencies.send_notification(
        command.job_id,
        command.completed_at,
    )
    connection.commit()
```

通常利用では`dependencies`を省略できる。Module Testでは明示的な`Dependencies`を渡せる。

Connectionは実行時の値として別に受け取っている。この例の`default()`はJob操作と通知実装を選ぶが、Connection Poolを生成しない。

## 6. `default()`をデフォルト引数へ直接置かない

Pythonのデフォルト引数は関数定義時に評価される。

```python
def execute(
    *,
    dependencies: Dependencies = Dependencies.default(),
):
    ...
```

では、ObjectがModule読込み時に生成され、呼び出し間で共有される。Connection、Client、mutableなFakeを含む場合は、予期しない共有や初期化時期になる。

`None`を受け、呼び出し時に`default()`を実行する。

```python
def execute(*, dependencies: Dependencies | None = None):
    if dependencies is None:
        dependencies = Dependencies.default()
```

この書き方はPythonの評価規則への対応であり、言語非依存の原則ではない。

## 7. Request側がConnectionを管理する形

Web FrameworkがRequest単位のConnectionまたはSessionを管理するなら、EntryからUseCaseへ渡せる。

```python
def handle(request, connection: Connection):
    command = CompleteJobCommand(...)
    execute(command, connection)
    return build_response(...)
```

責任は次のように分かれる。

```text
entry/web/app.py          # Poolを起動時に生成し、停止時に破棄する
entry/web/request.py      # Request単位でConnectionを取得し、返却する
feature/complete_job.py   # Connectionを使い、Transactionを確定する
```

この形では、UseCaseの`Dependencies`へPoolやConnection Factoryを入れる必要はない。UseCaseが必要とする完成したConnectionを渡す。

UseCaseのTransactionがRequestより短い、または一つのRequestで複数の独立Transactionを使う場合は、次節の取得能力を渡す形を検討できる。

## 8. UseCaseがConnection取得時点を決める形

UseCase自身がConnectionの取得と返却の範囲を決めるなら、取得能力を直接依存にできる。

```python
from contextlib import AbstractContextManager


class AcquireConnection(Protocol):
    def __call__(self) -> AbstractContextManager[Connection]:
        ...
```

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    acquire_connection: AcquireConnection
    complete_job: CompleteJob

    @classmethod
    def default(cls) -> Self:
        from application_runtime import acquire_connection
        from data_component.job import complete_job

        return cls(
            acquire_connection=acquire_connection,
            complete_job=complete_job,
        )
```

```python
def execute(
    command: CompleteJobCommand,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    with dependencies.acquire_connection() as connection:
        dependencies.complete_job(command.job_id, connection)
        connection.commit()
```

`application_runtime.acquire_connection`は、Application起動時に作られたPoolからConnectionを取得するProviderだとする。`Dependencies.default()`が呼ばれるたびにPoolを作るのではない。

この形なら、UseCaseはConnectionのscopeを決められる。一方、Runtimeが初期化済みであることへの前提が増え、通常構成の追跡と起動順序をTestする必要がある。

## 9. PoolのFactoryとConnectionのFactoryを混同しない

「Connection Factory」という名前が、次の二つを指すことがある。

```text
設定からConnection Poolを作るFactory
既存Poolから一回のConnectionを得るProvider
```

前者は長寿命Resourceを生成する。後者は共有Resourceから一回の利用を開始する。

```python
pool = create_pool(settings)       # Application起動時
connection = pool.acquire()        # RequestまたはUseCase単位
```

PoolのFactoryが設定、接続数、Timeoutを一括して持つなら、その責務はFactory自身にある。Composition RootからFactoryを呼ぶことによって、個別Moduleから同じFactoryを呼ぶ場合にはない設定能力が追加されるわけではない。

外側のCompositionへ置く主な理由は、Poolを一度だけ生成し、共有範囲を決め、起動時に失敗を検出し、停止時に破棄するためである。

## 10. Local Defaultで確定していること

Module近傍の`default()`には、次の性質がある。

- そのModuleが通常どの実装を選ぶかを近くに記述できる
- 呼び出し側は通常構成を毎回組み立てなくてよい
- Testでは明示的な`Dependencies`を渡せる
- 下位Moduleの標準構成を上位`Dependencies`へ入れずに済む
- 処理本体を、直接依存だけで実行できる

これらは、Module単体の処理を直接依存の差し替えによってTestできることを意味する。

次は自動的には保証しない。

- `default()`が正しいProduction実装を選んでいること
- すべてのRuntime設定が一貫していること
- PoolやClientのlifecycleが正しいこと
- Application全体の標準構成を一か所から一覧できること
- 複数のEntryで同じscopeになること

## 11. Module TestとDefault wiring Testを分ける

Module Testでは標準構成を使わず、直接依存を差し替える。

```python
def test_execute_commits_after_completion():
    connection = FakeConnection()
    calls: list[int] = []

    def fake_complete_job(job_id: int, connection: Connection) -> None:
        calls.append(job_id)

    dependencies = Dependencies(
        complete_job=fake_complete_job,
        send_notification=fake_notification,
    )

    execute(
        CompleteJobCommand(job_id=10, completed_at=fixed_time),
        connection,
        dependencies=dependencies,
    )

    assert calls == [10]
    assert connection.committed
```

このTestは、処理順序、直接利用、commitを確認できる。`Dependencies.default()`が正しい実装を選ぶことまでは確認しない。

Default wiringには別の確認が必要になる。

```python
def test_default_dependencies_can_be_built():
    dependencies = Dependencies.default()
    assert callable(dependencies.complete_job)
    assert callable(dependencies.send_notification)
```

このConstruction Testだけでは実Databaseとの接続やResource破棄まで保証しない。代表経路のIntegration TestまたはSmoke Testを別に持つ。

## 12. 外部Compositionを使う場合

呼び出し側の実行Contextによって、同じModuleへ異なる構成を渡す必要がある場合は、外側で構成を選べる。

```python
dependencies = Dependencies(
    complete_job=complete_job_default,
    send_notification=tenant_mail_sender,
)

execute(
    command,
    request_connection,
    dependencies=dependencies,
)
```

FrameworkがRequest scopeを管理する場合、Tenantや実行Modeが実装選択を決める場合、長寿命Resourceを共有する場合に意味がある。

外側で明示的に構成しても、Module localな`Dependencies`型と処理本体はそのまま使える。Local Defaultを持つことは、外側からの構成を禁止しない。

反対に、外側のCompositionが下位ModuleのRepositoryまで知り、すべての`Dependencies`を入れ子で組み立てると、上位が直接利用しない依存まで輸送しやすい。構成の一覧性と、Module境界の局所性を別に検討する。

## 13. Local Defaultは検証中の設計案である

Local Defaultは、長期運用で一般的に確立した方法として本書が保証するものではない。

特に次を検証する必要がある。

- Module数が増えたときに標準選択を追跡できるか
- 同じ設定や実装選択が複数の`default()`へ重複しないか
- PoolやClientの生成・破棄を一貫して扱えるか
- Application全体のwiringをどのTestで保証するか
- Web、CLI、Batchで異なるRuntime構成を説明できるか
- 実際の変更で局所性が維持されるか
- 障害時に通常構成を追跡できるか

小さなApplicationやOSSでドッグフーディングし、変更履歴、Test Setup、Resource管理、障害時の追跡性を観測して主張の強さを更新する。

## 14. 採用を判断する基準

Local Defaultを採用しやすい条件は次である。

- 通常実装がほぼ一つである
- Module単位のTest差し替えを短く書きたい
- 下位Dependencyを上位へ輸送したくない
- 長寿命Resourceのlifecycleを別の実行入口で扱える
- Construction Testと代表経路のIntegration Testを持てる

外側のCompositionを優先しやすい条件は次である。

- Runtime条件によって実装が変わる
- 同じResourceのscopeを複数Moduleで共有する
- 起動時にObject graph全体を検証したい
- FrameworkのDIやlifecycleへ統合する
- 構成差を一つの場所で比較する必要がある

両者は排他的ではない。Poolは外側で作り、Repositoryの通常実装はModule近傍で選ぶ、といった分担ができる。

## 15. この章の結論

Pythonでは、ModuleからPackageへの成長、`__init__.py`による公開面、Module localな`Dependencies`、呼び出し時の`default()`を組み合わせられる。

Module単体Testが可能であることと、Default wiringが正しいこと、長寿命Resourceのlifecycleが正しいことは別である。

Local Defaultは、通常実装の選択とTest差し替えを利用場所の近くへ置ける。一方、Connection Poolや長寿命Clientの生成・破棄、scope、起動時検証は、実行入口側のCompositionが扱う候補になる。

この分担は検証中の具体案である。選択、生成、lifecycle、受け渡しを一つの`default()`や一つのComposition Rootへ機械的にまとめず、実際の変更と運用で評価する。

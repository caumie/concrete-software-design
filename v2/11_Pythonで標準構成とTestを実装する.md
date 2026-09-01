# 第11章　Pythonで標準構成とTestを実装する

## この章の目的

本書の依存原則をPythonで実装する一つの方法として、Module localな`Dependencies`、`default()`、`Protocol`、ModuleからPackageへの成長を示す。Local Defaultの確定している性質と、運用上まだ検証が必要な性質を分ける。

## 1. この章は言語固有の具体例である

言語を越えて残る原則は次である。

- Moduleが直接利用する依存を明示する
- 下位Moduleの依存を上位へ輸送しない
- 標準構成とTest差し替えの範囲を分ける
- 実装選択、Resource生成、受け渡し経路を区別する

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

`job/__init__.py`で公開面を保つ。

```python
from .service import complete_job

__all__ = ["complete_job"]
```

利用側は引き続き次の形を使える。

```python
from data_component.job import complete_job
```

フォルダ化自体が前進なのではない。内部役割を分ける理由が具体化したためPackageへ進む。

## 3. Module localなDependenciesを定義する

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    connection: Connection
    complete_job: CompleteJob
    send_notification: SendNotification
```

これはApplication全体のDependency Objectではない。このModuleの直接依存だけを表す。

関数をDependencyとして受け取る場合、単純な形なら`Callable`で表現できる。Keyword-only引数や詳細な呼び出し契約を型検査したい場合は`Protocol`を使える。

```python
class CompleteJob(Protocol):
    def __call__(
        self,
        job_id: int,
        connection: Connection,
    ) -> None:
        ...
```

型を増やすこと自体を目的にしない。

## 4. `default()`で通常構成を作る

Productionで通常使う構成があるなら、次のように表現できる。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    connection: Connection
    complete_job: CompleteJob

    @classmethod
    def default(cls) -> Self:
        return cls(
            connection=connection_factory(),
            complete_job=complete_job_default,
        )
```

呼び出し側はDependenciesを省略できる。

```python
def execute(
    job_id: int,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    dependencies.complete_job(
        job_id,
        dependencies.connection,
    )
    dependencies.connection.commit()
```

## 5. `default()`をデフォルト引数へ直接置かない

Pythonのデフォルト引数は関数定義時に評価される。

```python
def execute(
    *,
    dependencies: Dependencies = Dependencies.default(),
):
    ...
```

では、Connection等がModule読込み時に生成され、呼び出し間で共有される可能性がある。

`None`を受け、呼び出し時に`default()`を実行する。この書き方はPythonの評価規則への対応であり、言語非依存の原則ではない。

## 6. Local Defaultで確定していること

Module近傍の`default()`には、次の性質がある。

- そのModuleが通常どの実装を選ぶかを同じ場所に記述できる
- 呼び出し側は通常構成を毎回組み立てなくてよい
- Testでは明示的なDependenciesを渡せる
- 下位Moduleの標準構成を上位Dependenciesへ入れずに済む

これは、Module単体の処理を直接依存の差し替えによってTestできることを意味する。

## 7. 単体TestとDefault wiringのTestを分ける

Module単体Testでは、標準構成を使わず直接依存を差し替える。

```python
dependencies = Dependencies(
    connection=fake_connection,
    complete_job=fake_complete_job,
)

execute(job_id=10, dependencies=dependencies)
```

このTestは処理順序やcommitを確認できる。しかし、`Dependencies.default()`が正しいProduction実装を組み立てていることまでは確認しない。

Default wiringには別の確認が必要になる。

- `default()`が必要なObjectを生成できるか
- 代表的な実行経路が通常構成で起動するか
- Databaseや外部ClientのResource lifecycleが正しいか
- 設定値や環境差が複数Moduleで食い違わないか

Construction Test、Smoke Test、Integration Test等で確認する。

## 8. 選択・生成・受け渡しを混同しない

`default()`が実装を選んでも、Connection Pool自体はApplication起動時に生成される場合がある。

外側からConnectionを渡しても、Repository実装の選択はJob Module近傍に残る場合がある。

Composition Rootを使うかLocal Defaultを使うかという二択だけでなく、次を個別に決める。

- 実装の選択理由を持つ場所
- Resourceを生成・破棄する場所
- 実行時の値を渡す経路

上から渡すこと自体を中央集権と呼ばず、近傍で選ぶこと自体を完全な局所化とも呼ばない。

## 9. 外部Compositionを使う場合

呼び出し側の実行Contextによって、同じModuleへ異なる構成を渡す必要がある場合は、外側で構成を選べる。

```python
dependencies = Dependencies(
    connection=request_connection,
    complete_job=complete_job_default,
)

execute(job_id=10, dependencies=dependencies)
```

FrameworkがRequest単位のConnection lifecycleを管理する場合や、Test以外のRuntime条件が構成を決める場合に意味がある。

Local Defaultと外部Compositionのどちらかを一般的な正解にしない。選択、生成、lifecycleの条件を具体的に書く。

## 10. Local Defaultは検証中の設計案である

Local Defaultは、長期運用で一般的に確立した方法として本書が保証するものではない。

特に次を検証する必要がある。

- Module数が増えたときに標準選択を追跡できるか
- 同じResourceの生成と破棄を一貫して扱えるか
- 設定変更が複数の`default()`へ重複しないか
- Application全体のwiringをどのTestで保証するか
- 実際の変更で局所性が維持されるか

小さなApplicationやOSSでドッグフーディングし、変更履歴、Test Setup、Resource管理、障害時の追跡性を観測して主張の強さを更新する。

## 11. 例: Module TestとWiring Test

Complete Job UseCaseについて、Testを二つに分ける。

```python
def test_execute_commits_after_completion():
    dependencies = Dependencies(
        connection=FakeConnection(),
        complete_job=fake_complete_job,
    )
    execute(job_id=10, dependencies=dependencies)
    assert dependencies.connection.committed
```

```python
def test_default_dependencies_can_be_built():
    dependencies = Dependencies.default()
    assert dependencies.connection is not None
    assert callable(dependencies.complete_job)
```

後者だけでは実Databaseとの接続やResource破棄まで保証しない。代表経路のIntegration Testを別に持つ。

## 12. この章の結論

Pythonでは、ModuleからPackageへの成長、`__init__.py`による公開面、Module localな`Dependencies`、呼び出し時の`default()`を組み合わせられる。

Module単体Testが可能であることと、Default wiringが正しいことは別である。Local Defaultは具体的な利点を持つ一方、長期運用、Resource lifecycle、全体wiringの検証が必要な実験的設計案として扱う。

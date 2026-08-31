# 第8章　TransactionをUseCaseで扱う

## 1. TransactionはDatabase機能であると同時に利用目的の意味を持つ

DatabaseのTransactionは技術的な仕組みである。

しかし、

> どこまでを一緒に成功させ、どこまでを一緒に失敗させるか

は、単なるRepositoryの都合では決められない。

例えばReschedule Jobという利用目的で、

- Jobの状態を確認する
- Scheduleを変更する
- 履歴を記録する

という三つの処理を一つとして扱いたい場合、その成功と失敗の単位を知っているのはUseCaseである。

具象設計論では、**Transaction境界をUseCase側で扱う**ことを基本とする。

---

## 2. ConnectionをUseCaseのDependencyとして持てる

UseCaseがTransactionを確定するなら、ConnectionはUseCaseが直接利用する依存である。

例えば、

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    conn: Connection
    complete_job: CompleteJob

    @classmethod
    def default(cls) -> Self:
        return cls(
            conn=connection_factory(),
            complete_job=complete_job_default,
        )
```

とする。

UseCaseは、

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
        dependencies.conn,
    )

    dependencies.conn.commit()
```

とできる。

`default()`が呼び出しごとにConnectionを生成するため、一つのUseCase実行に一つのConnectionを持てる。

---

## 3. Data Component側は渡されたConnectionを使う

Job Data Componentの公開関数では、同じConnectionを受け取る。

```python
# data_component/job/service.py

@dataclass(frozen=True, slots=True)
class Dependencies:
    repository: JobRepository

    @classmethod
    def default(cls) -> Self:
        return cls(
            repository=repository_default,
        )


def complete_job(
    job_id: int,
    connection: Connection,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    job = dependencies.repository.get(
        connection,
        job_id,
    )

    job.complete()

    dependencies.repository.save(
        connection,
        job,
    )
```

UseCaseのDependenciesとJob ServiceのDependenciesに親子関係はない。

UseCaseは、

- Jobの公開関数
- Connection

を直接利用する。

Job Serviceは、

- Repository

を直接利用する。

同じTransactionを共有するためにConnectionだけを引数として渡す。

---

## 4. Repositoryはcommitしない

Repositoryが自分でcommitすると、UseCaseが複数処理を一Transactionへまとめられなくなる。

望ましくない例は、

```python
def save(
    connection,
    job,
):
    connection.execute(...)
    connection.commit()
```

である。

Job保存後にSchedule保存が失敗しても、Jobだけが確定してしまう。

Repositoryは、

```python
def save(
    connection,
    job,
):
    connection.execute(...)
```

までに留める。

Transactionを確定するのはUseCaseである。

---

## 5. 複数Data Componentを一Transactionで扱う

Reschedule JobではJobとScheduleの両方を利用するとする。

UseCaseが利用する公開関数をDependencyとして持つ。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    conn: Connection
    ensure_reschedulable: EnsureReschedulable
    change_period: ChangePeriod

    @classmethod
    def default(cls) -> Self:
        return cls(
            conn=connection_factory(),
            ensure_reschedulable=ensure_reschedulable_default,
            change_period=change_period_default,
        )
```

そして、

```python
def execute(
    command: Command,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    dependencies.ensure_reschedulable(
        command.job_id,
        dependencies.conn,
    )

    dependencies.change_period(
        command.schedule_id,
        command.period,
        dependencies.conn,
    )

    dependencies.conn.commit()
```

とする。

Job側とSchedule側は、それぞれ自分のRepository依存を局所的に持つ。

UseCaseはその内部依存を知らない。

同じConnectionを渡すことで、一つのTransactionとして協調できる。

---

## 6. Data Component境界とTransaction境界は別である

JobとScheduleが別Data Componentであることは、

> 必ず別Transactionにしなければならない

ことを意味しない。

具象設計論では、

- Data Componentの境界
- Featureの境界
- UseCaseの境界
- Transactionの境界

を同一のものとして固定しない。

Data Componentは業務概念のまとまりである。

Featureは利用目的のまとまりである。

UseCaseはその実行順序を表す。

Transactionは成功・失敗を一つとして扱う範囲である。

これらは相互に関係するが、同じ理由では決まらない。

---

## 7. UseCaseをUseCaseの中へ入れ子にしない

UseCaseごとにTransaction境界を持たせるなら、UseCase同士をそのまま入れ子にすることは避ける。

例えば、

```python
def outer_usecase(...):
    first_usecase(...)
    second_usecase(...)
    connection.commit()
```

と書いても、`first_usecase()`自身がすでに`commit()`や`rollback()`を持っているなら、Outer UseCaseは全体を一つのTransactionとして扱えない。

UseCaseはそれぞれ、

> この利用目的では何を成功とし、何を失敗とするか

を持つ。

その成功・失敗条件を、別UseCaseから機械的に合成できるとは限らない。

そこで、より大きな利用目的が必要になった場合は、既存UseCaseを呼ぶのではなく、既存UseCaseが利用しているData Componentの公開関数など、Transactionを確定しない下位の処理を新しいUseCaseから組み合わせる。

```python
def larger_usecase(...):
    data_component_operation_a(...)
    data_component_operation_b(...)
    connection.commit()
```

この制約は多少の重複を生む可能性があるが、UseCaseごとのTransaction意味を局所的に保つために受け入れる。

---

## 8. UseCaseの外へTransactionを出す設計もあり得る

UseCase同士を組み合わせる必要が非常に強いなら、別の設計もある。

ConnectionをUseCaseの外から渡し、commit / rollbackをさらに外側のOrchestratorへ持ち上げる方法である。

```python
with transaction() as connection:
    usecase_a(
        command_a,
        connection=connection,
    )
    usecase_b(
        command_b,
        connection=connection,
    )
```

この形なら複数UseCaseを一つのTransactionへまとめやすい。

一方で、

- UseCase単体ではTransaction境界が完結しない
- 成功・失敗の意味が外側へ移る
- 呼び出し側がTransaction知識を持つ

という代償がある。

具象設計論の基本形はUseCaseごとにTransactionを考えることである。

しかし、**UseCaseの合成可能性を優先するならTransactionを外へ出す設計も選択肢になる**。どちらを採るかは、局所的なTransaction意味と合成可能性のどちらを重く見るかという設計判断である。

## 9. rollbackもUseCaseの単位に属する

明示的なTransaction制御が必要なら、

```python
def execute(
    command,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    try:
        dependencies.ensure_reschedulable(
            command.job_id,
            dependencies.conn,
        )

        dependencies.change_period(
            command.schedule_id,
            command.period,
            dependencies.conn,
        )

        dependencies.conn.commit()

    except Exception:
        dependencies.conn.rollback()
        raise
```

とできる。

Database APIやContext Managerがrollbackを自動化する場合もある。

実装方法が変わっても、

> Transactionの範囲はUseCaseの利用目的から決める

という考えは変わらない。

---

## 10. Transaction用ConnectionとRepositoryの標準構成を混同しない

Repositoryの標準構成は、

```python
repository=repository_default
```

としてほぼ固定できる。

一方、ConnectionはUseCase実行ごとに新しく必要になる実行資源である。

だからこそ、

```python
Dependencies.default()
```

を呼び出し時に実行し、

```python
conn=connection_factory()
```

とする意味がある。

Pythonで`Dependencies.default()`を関数定義時のデフォルト引数に置かない理由も、ここで具体的に現れる。

---

## 11. 複数Repositoryも同じConnectionを使う

一つのData Component内部でも、複数の永続化処理が必要になる場合がある。

また、UseCaseから複数Data Componentを利用する場合もある。

重要なのは、一つのTransactionに含めるRepositoryが同じConnectionを利用することである。

例えば、

```python
job_repository.save(
    connection,
    job,
)

history_repository.append(
    connection,
    history,
)
```

とする。

Repository自身がConnectionを生成したり、個別にcommitしたりすると、UseCaseがTransaction境界を制御できなくなる。

---

## 12. 外部I/Oは同じTransactionにはならない

Database更新後にメール送信を行うとする。

```python
dependencies.conn.commit()
dependencies.send_mail(...)
```

この場合、Databaseは成功してもメールが失敗する可能性がある。

順序を逆にして、

```python
dependencies.send_mail(...)
dependencies.conn.commit()
```

としても、メール送信後にDatabase commitが失敗する可能性がある。

Databaseと外部メール送信を、通常のACID Transaction一つとして扱うことはできない。

具象設計論は、この非対称性を抽象化で隠さない。

**Atomicにできないものを、Atomicであるかのように見せない。**

---

## 13. 必要になれば別の仕組みを追加する

Databaseと外部I/Oの整合性が本当に必要になったなら、

- Outbox
- Retry
- Event
- Compensation
- Idempotency

などを検討する。

しかし、これらを最初から全Featureへ用意しない。

問題が具体化した時点で、その問題を扱うための新しい責任として追加する。

この考えは第2章のブートストラップと同じである。

まだ存在しない複雑性のために、構造を先回りして増やさない。

---

## 14. Connection以外のTransaction表現も使える

具象設計論が要求しているのは`Connection`という具体的な型ではない。

利用するDatabaseライブラリによっては、

- Session
- Unit of Work
- Transaction object
- Context Manager

を使う方が自然な場合もある。

重要なのは、

> UseCaseが成功・失敗の範囲を判断できること

である。

PythonとDatabaseライブラリに合わせて、最も直接的な表現を選べばよい。

---

## 15. Unit of Workを導入する場合

複数RepositoryやTransaction制御をまとめるためにUnit of Workを導入することもできる。

ただし、

> DDDだからUnit of Workを作る

のではない。

ConnectionやSessionを直接扱うだけで十分なら、そのままでよい。

Transaction制御が複雑になり、

- commit / rollbackの共通処理
- 複数Repositoryの生成
- Session lifecycle

などに独立した意味が生じたときに、Unit of Workという役割へ具体化する。

---

## 16. Transactionのテスト

UseCase単体をテストするときには、Fake ConnectionをDependencyへ渡せる。

```python
class FakeConnection:
    def __init__(self):
        self.committed = False
        self.rolled_back = False

    def commit(self):
        self.committed = True

    def rollback(self):
        self.rolled_back = True
```

そして、

```python
dependencies = Dependencies(
    conn=FakeConnection(),
    ensure_reschedulable=fake_ensure,
    change_period=fake_change,
)
```

とする。

これによって、

- 正常時にcommitされる
- 途中で失敗したらrollbackされる
- 必要な処理順序になっている

ことをUseCase単体で確認できる。

DatabaseそのもののTransaction動作はIntegration Testで別に確認する。

---

## 17. UseCase Transactionは局所性と合成可能性のTrade-offである

UseCaseごとにTransactionを完結させる基本形では、

- 成功・失敗の意味をUseCaseの中で読める
- commit / rollbackの責任が局所化する
- Module単体でTransaction条件をTestしやすい

という利点がある。

一方、UseCase自身がTransactionを確定するため、複数UseCaseをそのまま一つのTransactionへ入れ子にしにくい。

合成可能性を優先するなら、Connectionとcommit / rollbackをUseCaseの一段上へ持ち上げてもよい。

```python
with transaction() as connection:
    usecase_a(
        command_a,
        connection=connection,
    )
    usecase_b(
        command_b,
        connection=connection,
    )
```

この場合は、Transactionの成功・失敗をUseCase単体ではなく外側のOrchestratorが決める。

したがって本書の基本形は、

> UseCaseごとにTransactionを考える

であるが、絶対規則ではない。

**Transaction意味の局所性と、UseCaseの合成可能性のどちらを優先するかというTrade-offとして設計する。**

## 18. この章の判断原則

Transactionについて、具象設計論では次のように考える。

1. 基本形ではTransaction境界を利用目的を知るUseCaseで扱う
2. UseCaseの合成可能性を優先する場合は、Transactionを一段上のOrchestratorへ持ち上げてよい
3. この選択はTransaction意味の局所性とUseCase合成可能性のTrade-offである
4. ConnectionはUseCaseが直接利用する依存として持てる
5. `default()`で呼び出しごとにConnectionを生成できる
6. Data Component側へ同じConnectionを引数として渡す
7. Repositoryはcommitしない
8. Data Component境界とTransaction境界を同一視しない
9. 外部I/Oを無理に一Transactionに見せない
10. Unit of Work等は必要性が具体化したときだけ導入する
11. Transaction制御もUseCase単体テストで差し替え可能にする

ここまでで、配置・公開・Feature・Data Component・Dependency・標準構成・Transactionという、具象設計の基本的な書込み側の構造が一通りつながった。

次章では、これらと異なる性質を持つ読み取り処理、Queryの配置を扱う。
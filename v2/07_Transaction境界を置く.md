# 第7章　Transaction境界を置く

## この章の目的

Data ComponentやRepositoryの配置とは別に、どこまでを一緒に成功させ、どこまでを一緒に失敗させるかを決める。UseCaseを基本のTransaction境界としつつ、合成が必要な場合の選択肢と外部I/Oの制約を示す。

## 1. Transactionは利用目的の成功範囲を表す

Database Transactionは技術的な機能である。しかし、何を一つの成功として扱うかは利用目的によって決まる。

Reschedule Jobで次の処理が必要だとする。

- Jobが再予定可能か確認する
- Scheduleを変更する
- 変更履歴を記録する

この三つを一緒に成功・失敗させたいなら、Reschedule Job UseCaseがその範囲を扱う候補になる。

## 2. UseCaseがConnectionを直接利用する

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    connection: Connection
    ensure_reschedulable: EnsureReschedulable
    change_period: ChangePeriod
    append_history: AppendHistory
```

```python
def execute(command, *, dependencies: Dependencies) -> None:
    dependencies.ensure_reschedulable(
        command.job_id,
        dependencies.connection,
    )
    dependencies.change_period(
        command.schedule_id,
        command.period,
        dependencies.connection,
    )
    dependencies.append_history(
        command.job_id,
        command.period,
        dependencies.connection,
    )
    dependencies.connection.commit()
```

Connectionは、UseCaseがTransactionを確定するために直接利用する依存である。

## 3. Repositoryはcommitしない

Repositoryが個別にcommitすると、UseCaseは複数の変更を一つにまとめられない。

```python
def save(connection, job):
    connection.execute(...)
    connection.commit()  # UseCaseのTransactionを分断する
```

Repositoryは取得と保存までを行い、Transactionの確定はその利用目的を知る場所へ残す。

```python
def save(connection, job):
    connection.execute(...)
```

## 4. Data Component境界とTransaction境界を一致させない

JobとScheduleが別Data Componentでも、同じConnectionを渡せば一つのTransactionで変更できる。

反対に、同じJob Data Componentを使うCreate JobとComplete Jobが、別々のTransactionになることもある。

- Data Componentは、意味あるデータを中心にした配置
- Featureは、利用目的のまとまり
- Transactionは、成功と失敗の範囲

これらは関係するが、同じ理由では決まらない。

## 5. UseCaseをそのまま入れ子にしない

各UseCaseが自分でcommitする場合、別UseCaseから呼ぶと全体を一つのTransactionへまとめられない。

```python
def larger_usecase(...):
    first_usecase(...)   # 内部でcommit
    second_usecase(...)  # 内部でcommit
```

より大きな利用目的が必要なら、既存UseCaseが利用しているData Componentの公開操作を新しいUseCaseから組み合わせられる。

```python
def larger_usecase(..., connection):
    operation_a(..., connection)
    operation_b(..., connection)
    connection.commit()
```

多少の処理列の重複が生じても、各UseCaseの成功条件を混同しないことを優先する場合がある。

## 6. 合成を優先するなら一段上へ持ち上げる

複数UseCaseを一つのTransactionへ組み合わせる要求が強いなら、Transactionを外側のOrchestratorへ持ち上げられる。

```python
with transaction() as connection:
    usecase_a(command_a, connection=connection)
    usecase_b(command_b, connection=connection)
```

この設計では、UseCase単体で成功・失敗が完結しない。呼び出し側がTransaction知識を持つ。

UseCase内と外側のどちらが常に正しいわけではない。利用目的ごとの局所性と、複数UseCaseの合成可能性のどちらを必要とするかで決める。

## 7. rollbackも同じ境界で扱う

```python
def execute(command, *, dependencies: Dependencies) -> None:
    try:
        dependencies.ensure_reschedulable(...)
        dependencies.change_period(...)
        dependencies.connection.commit()
    except Exception:
        dependencies.connection.rollback()
        raise
```

Database APIやContext Managerがrollbackを自動化する場合もある。実装方法が変わっても、どの処理を一緒に失敗させるかは明示する。

## 8. 外部I/Oを同じTransactionに見せない

Database更新とMail送信は、通常のACID Transaction一つにはならない。

```python
connection.commit()
send_mail(...)
```

では、commit後にMailが失敗し得る。

```python
send_mail(...)
connection.commit()
```

では、Mail送信後にcommitが失敗し得る。

PortやAdapterで包んでも、この物理的制約は消えない。必要になった時点でOutbox、Retry、Idempotency、Compensationなどの責任を追加する。

## 9. TransactionをTestする

UseCase単体では、Fake Connectionを渡して処理順序とcommit・rollbackを確認できる。

```python
class FakeConnection:
    def __init__(self):
        self.committed = False
        self.rolled_back = False

    def commit(self) -> None:
        self.committed = True

    def rollback(self) -> None:
        self.rolled_back = True
```

Database自体のIsolationやrollback動作はIntegration Testで別に確認する。

## 10. この章の結論

TransactionはDatabaseの配置ではなく、利用目的の成功と失敗の範囲から決める。

基本形ではUseCaseがConnectionを直接利用し、Repositoryはcommitしない。複数UseCaseの合成を優先する場合は、Transactionを一段上へ持ち上げられる。外部I/Oを同じAtomicな処理に見せず、必要な整合性責任を別に設計する。

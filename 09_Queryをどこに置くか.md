# 第9章　Queryをどこに置くか

## 1. Queryは「層」ではなく役割である

書込み処理では、Feature、Data Component、UseCase、Transactionの関係を中心に考えてきた。

一方、一覧、検索、集計、表示用データ取得などの読み取り処理は、書込みとは異なる性質を持つ。

ここで、

```text
query/
```

というトップレベルの層を必ず作る必要はない。

Queryは、

> 状態を変更せず、必要な情報を読み取る

という役割を表す語である。

配置を決めるのはQueryという役割ではなく、

> **その読み取りが何のために存在するか**

である。

---

## 2. Feature固有のQuery

Complete Jobを実行する前に、利用者へ確認画面を表示するための情報が必要だとする。

このQueryがComplete Jobにしか意味を持たないなら、

```text
feature/
    complete_job/
        usecase.py
        query.py
```

と置ける。

```python
# feature/complete_job/query.py

def load_confirmation(
    job_id: int,
    connection,
) -> ConfirmationData:
    row = connection.execute(
        """
        SELECT id, name, status
        FROM jobs
        WHERE id = ?
        """,
        (job_id,),
    ).fetchone()

    return ConfirmationData(
        job_id=row["id"],
        name=row["name"],
        status=row["status"],
    )
```

このQueryを別のFeatureが利用していないなら、Featureの外へ移す理由はない。

---

## 3. Queryだけで一つのFeatureが成立する

一覧表示そのものが一つの利用目的であることもある。

例えば、

```text
feature/
    list_jobs/
        query.py
```

とする。

書込みUseCaseが存在しなくてもよい。

```python
def execute(
    condition: SearchCondition,
    *,
    dependencies: Dependencies | None = None,
) -> list[JobListItem]:
    ...
```

読み取りだけのFeatureであっても、利用目的として独立していればFeatureとして成立する。

具象設計論では、

> Featureには必ずUseCaseクラスが必要

とは考えない。

---

## 4. 画面用Queryも利用目的の配下に置く

Dashboard表示にしか意味を持たない集計を考える。

```text
feature/
    dashboard/
        query.py
```

と置ける。

例えば、

```python
def load_dashboard(
    user_id: int,
    connection,
) -> DashboardData:
    ...
```

が、

- Dashboard表示のために存在する
- Dashboard固有の表示単位で集計する
- 他の利用目的から再利用する意味がない

のであれば、Dashboard Featureの配下に置くと意味を説明しやすい。

Webから呼ばれることだけを理由に、Databaseへ触れるQueryをEntry配下へ置かない。EntryはRequestを内部値へ変換し、Featureの結果をResponseへ変換する。SessionやCookieなどEntry境界そのものを読む処理なら、`entry/web`内の意味ある概念へ置ける。

---

## 5. Data Component内部で必要なQuery

Data Componentの公開処理を成立させるために、補助的な読み取りが必要な場合もある。

例えばJob Data Component内部で、状態遷移に必要なJob本体を取得する処理がある。

それは、

```text
data_component/
    job/
        repository.py
```

のような役割としてJob配下に置ける。

つまり、読み取り処理だからQueryと呼ぶ必要すらない場合もある。

名前は、その処理の意味を最もよく表すものを選ぶ。

---

## 6. Domain Modelを読み取りのためだけに復元しない

一覧表示で100件のJobを表示したいとする。

必要なのは、

- Job ID
- Name
- Status
- Planned Date

だけかもしれない。

このとき100個の完全なJob Domain Modelを復元し、

```python
jobs = repository.find_all(...)
items = [
    JobListItem(
        ...
    )
    for job in jobs
]
```

とする必要はない。

Queryが直接、

```python
def execute(
    condition,
    connection,
) -> list[JobListItem]:
    rows = connection.execute(...)

    return [
        JobListItem(
            job_id=row["id"],
            name=row["name"],
            status=row["status"],
            planned_date=row["planned_date"],
        )
        for row in rows
    ]
```

と結果を作ってよい。

Domain Modelは、状態変更や業務規則を表現するために価値がある。

読み取りの都合だけで、そのコストを常に支払う必要はない。

---

## 7. Read Modelは利用側に必要な結果の形でよい

Queryの結果は、利用側が必要とする形に具体化する。

```python
@dataclass(frozen=True, slots=True)
class JobListItem:
    job_id: int
    name: str
    status: str
```

これはDomain Entityの代用品ではない。

List Jobs Featureが必要とするRead Modelである。

別のDashboardでは、

```python
@dataclass(frozen=True, slots=True)
class DashboardJob:
    job_id: int
    delay_days: int
    warning: bool
```

のように、異なる形を持ってよい。

同じDatabase行を読むからといって、一つのDTOへ統一する必要はない。

---

## 8. SQLを直接使うことを過剰に避けない

Queryは、読み取り対象を効率よく取得するためにSQLと近くなることがある。

```python
rows = connection.execute(
    """
    SELECT
        status,
        COUNT(*) AS count
    FROM jobs
    GROUP BY status
    """
)
```

この処理を、

```text
Repository
→ Domain Entity
→ Domain Service
→ DTO
```

と何段も経由させる理由がなければ、直接Queryとして書いてよい。

Queryの責任は、

> 必要な読み取り結果を作ること

である。

Databaseの構造がQueryの意味へ強く影響する場合は、その依存を受け入れた上で局所化する。

---

## 9. QueryにもDependencyを局所的に定義できる

QueryモジュールがConnectionや外部Data Sourceを直接利用するなら、そのモジュールで依存を定義する。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    conn: Connection

    @classmethod
    def default(cls) -> Self:
        return cls(
            conn=connection_factory(),
        )
```

```python
def execute(
    condition: SearchCondition,
    *,
    dependencies: Dependencies | None = None,
) -> list[JobListItem]:
    if dependencies is None:
        dependencies = Dependencies.default()

    ...
```

書込みUseCaseと同様に、Query側のDependenciesもそのモジュールだけのものである。

他FeatureやData ComponentのDependenciesを入れ子にしない。

---

## 10. Queryの重複を早期に共通化しない

二つのQueryに、

```sql
SELECT id, name, status FROM jobs
```

という同じSQLが現れたとする。

これだけで、

```text
shared/
    job_query.py
```

へ移す必要はない。

一方はComplete Job確認用、もう一方はSearch Result用かもしれない。

現時点で同じ列を読んでいるだけで、変更理由は異なる可能性がある。

共通化するのは、

> 実装が同じだから

ではなく、

> 同じ読み取り概念として扱う意味があるから

である。

---

## 11. Queryの共通概念が見えた場合

後から複数Featureで、

> Jobの検索条件、Page処理、並び順、結果の意味が本当に共通している

と分かったなら、独立した概念として切り出すことを検討できる。

例えば、

```text
feature/
    search_jobs/
```

が一つの読み取り機能として成立するかもしれない。

あるいは、業務上の独立したRead Modelが見えれば別の名前を与えてもよい。

具象設計論は、Query共有のためにあらかじめ共通層を作らない。

具体例から共通概念が見えてから作る。

ただし、`shared/`へ置くことを禁止する規則ではない。広い範囲で共有する意味を意図して選んだなら、その判断を公開面と名前に表す。

---

## 12. 書込みUseCaseの内部Query

UseCaseが状態変更のために情報を読む場合、その読み取りをすべて独立Queryへ分ける必要はない。

例えば、

```python
def execute(*args, **kwargs):
    row = dependencies.conn.execute(...).fetchone()
    ...
```

だけで十分なら、そのままでもよい。

読み取り処理が、

- 複雑になった
- 独立してテストしたい
- 同じFeature内で複数箇所から使う
- 名前を与えた方が意味が明確になる

といった理由が現れたときに`query.py`へ分ける。

---

## 13. QueryとTransaction

単純な読み取りでは、明示的なcommitは通常必要ない。

しかし、

- 一貫したSnapshotを読む
- Lockを取得する
- 書込みUseCaseの途中で読み取る

など、Transactionの意味を持つ場合もある。

このときも、

> QueryだからTransactionを持たない

という機械規則にはしない。

利用目的から必要な整合性を考える。

---

## 14. Queryの配置を決める問い

Queryについては、次の順序で考える。

1. この読み取りは何のために存在するか
2. 特定Featureだけに意味を持つか
3. 画面や利用経路ではなく、どの利用目的に属するか
4. Entry境界そのものを読む処理か
5. 読み取り自体が一つのFeatureか
6. Domain Modelを復元する必要が本当にあるか
7. 利用側が必要な結果の形は何か
8. 同じSQLではなく、同じ意味として共有できるか
9. 依存はQueryモジュール自身で定義できているか

Queryは、具象設計論の「役割と概念を分ける」という考え方が特に表れやすい領域である。

次章では、WebやCLIなど外部からシステムへ入る入口を扱う。

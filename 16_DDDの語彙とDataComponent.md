# 第16章　DDDの語彙とData Component

## 1. DDDの語彙は意味が合う場所で使う

Domain-Driven Designには、Ubiquitous Language、Entity、Value Object、Aggregate、Repository、Domain Service、Domain Event、Bounded Contextなど、業務概念をコードへ落とし込むための豊富な語彙がある。

具象設計論では、これらの語彙を必要な場所でそのまま利用する。

重要なのは、用語の知名度ではなく、その用語が現在の責任を正確に説明できることである。

## 2. Ubiquitous Languageを名前へ反映する

業務上の言葉を、Feature、Data Component、公開関数、型の名前へ反映する。

```text
data_component/
    job.py

feature/
    complete_job.py
```

のように、コードの配置から意味を読み取れる状態を作る。

一般的な`service`や`manager`より、業務上の具体的な語が使えるならそちらを優先する。

## 3. EntityやValue Objectは必要になったときに導入する

JobId、Money、Periodなどに独立した不変条件や型としての意味があるなら、EntityやValue Objectとして表現できる。

```python
@dataclass(frozen=True, slots=True)
class JobId:
    value: int
```

単純なPrimitiveで十分なら、そのまま使う。

役割名を先に揃えるのではなく、具体的な意味が現れたときにDDDの語彙を使う。

## 4. Data Componentはコード配置上の意味単位である

本書のData Componentは、

- 意味あるデータ
- 状態
- 規則
- Repository
- 公開操作

を近くにまとめるためのコード配置上の単位である。

```text
data_component/
    job/
        __init__.py
        domain.py
        repository.py
```

一つのData Componentの中に、複数のEntityやValue Objectが存在してよい。

## 5. DDD Aggregateは整合性境界として扱う

DDDでAggregateという語を使う場合は、Aggregate Rootを通じて整合性を守る単位として扱う。

Data Componentはコード配置、DDD AggregateはDomain Model上の整合性という観点からそれぞれ定義する。

一つのData Componentが一つのAggregateを中心にする場合も、複数Aggregateを含む場合もある。

この区別によって、

- Data Componentはコード配置
- AggregateはDomain Model上の整合性境界

として、それぞれの判断を独立して扱える。

## 6. RepositoryはData Componentの近くへ置ける

RepositoryがJobというデータ概念を扱うなら、

```text
data_component/
    job/
        repository.py
```

へ置ける。

DDD AggregateをRepository単位として採用する場合は、そのRepositoryがAggregate Rootを単位として取得・保存するという制約を追加できる。

配置上の意味と、Repository Patternとしての契約を重ね合わせて使う。

## 7. Domain Serviceは具体的な責任へ名前を与える

Entity一つに置くより、複数のDomain Objectを使う業務規則として独立した方が分かりやすい場合は、Domain Serviceという考え方を使える。

ただしファイル名は、

```text
pricing.py
business_day.py
allocation.py
```

のように、具体的な意味を表す名前を優先してよい。

## 8. Domain Eventは事実を表す

業務上起きた事実を他の処理へ伝える必要がある場合、Domain Eventを使える。

```text
JobCompleted
ScheduleChanged
```

のように、処理命令ではなく起きた事実を名前にする。

Event配送や非同期処理の責任は別途設計する。

## 9. Bounded Contextは言葉の意味が変わる境界として扱う

同じ`Job`という語が、部門やシステムごとに異なる意味を持つ場合がある。

その差が大きくなれば、Bounded Contextとして境界を設計する。

Data ComponentやFeatureの増加は、Context境界を見直す材料にもなる。

## 10. DDDの語彙をコード構造へ機械的に展開しない

DDDの用語は、必要な概念を説明するために使う。

```text
entity/
value_object/
repository/
domain_service/
event/
```

という役割は、具体的な概念の内部で必要になった時点で配置する。

Jobという意味の配下に、

```text
data_component/
    job/
        domain.py
        repository.py
```

と置く方が、そのプロジェクトで変更を追いやすいならその形を使う。

## 11. Domain理解が変われば構造も更新する

最初はJob Data Componentの中に置いたScheduleが、独立した状態とライフサイクルを持つと分かったなら、

```text
data_component/
    job/
    schedule/
```

へ分けられる。

逆に、別々に扱っていた概念が一つの整合性境界だと分かれば、DDD Aggregateとして再設計することもできる。

DDDの語彙を、具体化によって得られたDomain理解を表現するために使う。

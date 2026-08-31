# 第21章　AI Agentへ具象設計を与える

## 1. Agentには完成形より判断規則を与える

AI Agentへ、

```text
data_component/
feature/
entry/
```

だけを与えると、その構造をテンプレートとして機械的に再現する可能性がある。

具象設計論をAgentへ与えるときに重要なのは、

> なぜその構造なのか。

という判断規則である。

---

## 2. 最上位の原則

Agentへ最初に与える原則は次である。

> 現在説明できる概念だけを構造として作る。
>
> 役割だけを理由に新しいトップレベル層を作らない。
>
> 新しい意味が見えたら、新しい概念の追加・分割・統合を検討する。

---

## 3. 配置規則

Agentには、コードを追加する前に次を確認させる。

1. このコードは何についてのコードか
2. 既存のどの概念の配下なら意味を説明できるか
3. 親概念の意味を越えないか
4. 一つのModuleで表現できる概念を、先にPackage化していないか
5. 新しいフォルダを作るだけの具体的意味があるか
6. Repository、Query等の役割名だけで配置を決めていないか

一つのFeatureやData Componentが一つのModuleで十分なら、

```text
feature/complete_job.py
data_component/job.py
```

から始める。

内部に複数の役割が具体化したときだけ、

```text
feature/complete_job/
data_component/job/
```

というPackageへ成長させる。

PythonではPackage化後に`__init__.py`で公開面を維持できるため、利用側のimportを保ったまま内部を細分化できる。

## 4. 公開面の規則

別概念から利用する場合は、原則として概念トップの公開関数・公開型を利用する。

```python
from data_component.job import complete_job
```

を優先する。

```python
from data_component.job.internal.xxx import ...
```

が必要なら、その理由を確認する。

深いimportを機械的に禁止するのではなく、公開面不足や概念境界の問題を疑う。

---

## 5. Dependency規則

Pythonでは、各モジュールが直接使う依存だけを定義する。

どのModuleがどの公開要素をimportし、利用しているかを依存グラフとして扱う。この規則はUseCaseだけでなく、Data Component、Entry、Query、外部サービスにも適用する。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    ...

    @classmethod
    def default(cls) -> Self:
        ...
```

Dependencyを階層化しない。

配下モジュールのDependencyを上位へ輸送しない。

通常実行では`default()`を使う。

Module単体Testでは、そのModuleの直接依存だけを差し替える。依存先Moduleのさらに下位の依存まで同じTestへ持ち込まない。

---

## 6. PythonのUseCase基本形

```python
def execute(
    require_arg: int,
    *,
    dependencies: Dependencies | None = None,
):
    if dependencies is None:
        dependencies = Dependencies.default()

    ...
```

`Dependencies.default()`を関数のデフォルト引数へ直接書かない。

Pythonではデフォルト引数が関数定義時に評価されるためである。

---

## 7. 差し替える範囲

差し替え口は、主としてModule単体Testと、実際に複数構成が必要な箇所のために持つ。

Productionの標準実装が固定なら`default()`で直接使う。

---

## 8. Repository規則

Repositoryは対象となる概念の近くへ置く。

```text
data_component/
    job/
        repository.py
```

を基本例とする。

Repositoryが対象概念から独立した意味を持つ場合は、その意味に応じて別概念として配置できる。

---

## 9. Transaction規則

UseCaseがTransactionを扱う。

Repositoryはcommitしない。

一つのUseCaseで複数Data Componentを変更する場合、同じConnectionを渡して一Transactionにできる。

UseCase自身がTransaction境界を持つ基本形では、UseCaseを別UseCaseの中へそのまま入れ子にしない。より大きな利用目的では、Data Componentの公開処理等を組み合わせて新しいTransaction境界を作る。

UseCase合成を優先するプロジェクトでは、ConnectionとTransaction境界をUseCase外へ持ち上げる設計も選択肢になる。その場合は、UseCase単体で成功・失敗が完結しなくなることを明示する。

Data Component境界とTransaction境界を機械的に一致させない。

## 10. Query規則

Queryは層ではなく役割である。

読み取り自体が利用目的なら、まず`feature/list_jobs.py`のような一つのFeature Moduleとして作れる。内部役割が増えたときだけPackage化する。

Databaseへ触れる画面用Queryも、DashboardやList Jobs等の利用目的を表すFeatureへ置く。Webから呼ばれることだけを理由にEntry配下へ置かない。SessionやCookie等のEntry境界そのものを読む処理は`entry/web`内の意味ある概念へ置ける。

Read ModelのためにDomain Modelを必ず復元する必要はないが、Domain Logicを利用して結果を作る意味があるならDomain Modelを復元してよい。

書込みUseCase内の読み取りを独立Queryとして積極的に切り出さない。Data Componentが存在するなら、変更対象の取得・保存はData Component/Repository側へ寄せることを優先する。

## 11. 外部サービス規則

外部APIやSDKを利用するコードについて、必ずFeature内部へ置くとも、必ず`infrastructure/`へ置くとも判断しない。

次を確認する。

- 外部サービス固有のAuthenticationがあるか
- SDK型や外部Responseの変換があるか
- Timeout / Retry / Rate Limit制御があるか
- 外部Service固有のError処理があるか
- 複数Featureから同じ外部境界を利用する見込みが明確か

これらが最初から十分に見えているなら、

```text
external_service/
    stripe.py
```

などのトップレベル概念を作ってよい。

単純なSDK呼び出しだけなら、まず利用する意味の近くへ置いてよい。

> まずは利用する意味の近くへ置き、具体例を観測する。

を基本としつつ、すでに明確な外部境界を無理に局所Featureへ押し込まない。

## 12. Event規則

直接呼び出しを基本にする。

Eventを使う場合は、

- 起きた事実として公開する意味
- ProducerがConsumerを知らない価値
- 配送責任を追加するコスト

を説明する。

「疎結合だから」だけでEventを導入しない。

---

## 13. 新概念は意味を確認して追加する

Agentは一般的なArchitecture Patternを知っているため、

```text
ports/
adapters/
services/
factories/
interfaces/
```

を先回りして作ることがある。

そのため、

> 新しい役割・フォルダは、現在のコードで独立した責任を説明できる場合だけ作る。

と明示する。

---

## 14. 新しい具体例から既存構造を見直す

一方、

> 既存構造を絶対に変更しない

とも指示しない。

新しい具体例によって、

- 親概念の意味を越えた
- 同じ規則が複数箇所へ現れた
- 公開面が巨大化した
- 常に複数概念を同時修正する

といった兆候があれば、構造変更候補を提示させる。

複数概念を同時修正した場合は、同時変更だけを統合理由にしない。最初に変わった依存先の意味と、その公開面を利用するimport関係を特定させる。

---

## 15. 通常適用する規則

Agent向け規約として、通常は次を適用する。

- 既存概念の意味を確認してから配置する
- 他概念からは公開面を優先して利用する
- Dependencyは直接利用するモジュールで定義する
- Pythonのimportと直接利用を依存グラフとして扱う
- Pythonでは標準構成を`default()`で表現する
- Module単体Testでは直接依存だけを差し替える
- Repositoryはcommitしない
- UseCaseがTransaction境界を決める
- 新しい構造を追加したら周辺概念との重複を確認する
- Test可能性を維持する

---

## 16. 理由なく行わないこと

- 役割名だけを理由にトップレベル層を追加しない
- Dependencyを階層化しない
- 巨大なApplicationDependenciesを作らない
- Repository一つのために`infrastructure/`階層を作らない
- 利用箇所数だけでSharedへ移さない
- 全依存を将来のためにInterface化しない
- QueryだからQuery層へ置く、と判断しない
- 疎結合という理由だけでEvent化しない
- 既存フォルダ構造をArchitectureの正解とみなさない

これらは機械的な禁止ではない。Runtime切替、意図したShared、Framework制約などの理由がある場合は、その理由と影響範囲を明示して選択できる。

---

## 17. Agentの実装後Review

実装後にAgentへ次を確認させる。

### 配置

- 新しいファイルは親概念の意味を越えていないか
- 新しいフォルダは具体的な意味を持つか

### 依存

- 直接利用しないDependencyを持っていないか
- 配下モジュールのDependencyを輸送していないか
- Pythonのimportと直接利用から依存関係を追えるか
- Testで直接依存だけを差し替えているか

### 概念

- 既存Featureとの意味重複はないか
- Data Componentへ移すべき業務規則はないか
- 既存Data Componentを分割する材料は増えていないか

### 境界

- 深いimportが増えていないか
- Framework型が不要な場所へ漏れていないか

---

## 18. Agentに自動Refactoringさせるか

検出した問題を自動で修正させるかはプロジェクト方針による。

大きな概念変更は、

> 候補と理由だけ提示する

方が安全な場合もある。

例えば、

```text
ScheduleをJob Data Componentから分割する候補
理由:
- 独立した状態変更が3Featureで発生
- Job完了後もSchedule変更が存在
```

のように提案させる。

概念発見を完全自動化しないという本書の立場と一致する。

---

## 19. Agent向け文書の構成

GitHub等へAgent用規約を置く場合、

```text
docs/
    principles/
    examples/
    python/
```

のように分けてもよい。

ただし、この構造自体も必要性に応じて決める。

重要なのは、

- 中心原則
- 通常適用する規則
- 既定から外れるときに説明する条件
- 正例
- 反例
- Python固有実装
- Review観点

をAgentが参照できることである。

---

## 20. 境界判断の例を与える

Agentは正例から似たコードを生成できる。

境界を理解させるには、配置を見直す例も役立つ。

例えば、

```text
見直す例:
feature/complete_job/
    infrastructure/
        repository.py

理由:
RepositoryはComplete Job固有ではなくJob概念の永続化である。
```

または、

```text
見直す例:
CompleteJobDependencies(
    job=JobDependencies(...)
)

理由:
Dependency階層を作り、下位依存を上位へ輸送している。
```

のように示す。

---

## 21. Agentに与える最短の要約

具象設計論をAgent向けに短くまとめるなら、次のようになる。

> コードを役割だけで分類せず、現在説明できる具体的な概念の配下へ置く。
> 必要な役割だけを作る。
> 他概念からは公開面を利用する。
> 依存はPythonのimportと直接利用の関係として捉え、利用するModuleだけで明示し、標準構成を局所的に持つ。
> TestではそのModuleの直接依存だけを差し替える。
> 概念は固定せず、具体例が増えたら分割・統合を再評価する。

これがAgentに模倣させたい具象設計の最小核である。

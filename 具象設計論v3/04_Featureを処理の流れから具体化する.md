# 第4章　Featureを処理の流れから具体化する

## この章の目的

前章で作ったComplete Job Featureを使い、利用目的が完了するまでの処理を同じ抽象度で並べる。そこから、Featureが直接利用する概念、Feature自身が持つ判断、分岐、Loop、Transactionの候補を具体化する。

処理列のコメントは、本書の必須構文ではない。頭の中にある利用目的をコードへ置く前に外へ出し、配置と依存を検討するための手段である。判断結果は最終的に、関数、Module、Package、公開面、依存関係へ反映する。

## 1. Featureは利用目的に名前を与える

Featureは、利用者またはシステムが達成しようとする目的をまとめる。

```text
feature/
    create_job.py
    complete_job.py
    reschedule_job.py
```

`complete_job.py`という名前は、そこに置かれたコードが何のために存在するかを示す。単にApplication層に属するコードを集めた`service.py`より、変更の目的を調べる起点になりやすい。

Featureは、Application層の別名でも、すべてのコードを閉じ込める最終単位でもない。

- 複数Featureに同じ状態と規則が現れれば、Data Componentを検討する
- Entryがなければ成立しない処理は、Entryの配下に置ける
- 外部サービスとの通信、認証、変換、失敗制御が独立した意味を持てば、別の兄弟概念を作れる
- 複数Transactionと待機をまたぐ状態が現れれば、長期Processを検討できる

Featureは利用目的を具体化する初期単位であり、すべてをVertical Sliceへ固定する規則ではない。

## 2. 完了までの処理列を先に書く

Complete Jobを実装する前に、何をもって一連の処理が完了するかをコメントで並べる。

```python
def execute(...):
    # 操作者がJobを完了できることを確認する

    # Jobを完了させる

    # 完了通知の依頼を記録する

    # 一連の変更を確定する
```

コメント一行は、UseCaseから見た一つの処理単位である。各コメントの下に、複数行の実装や別概念の公開操作が置かれ得る。

これは、コード一行をそのまま日本語に置き換えるコメントとは異なる。

```python
# complete_job関数を呼ぶ
complete_job(command.job_id, connection)
```

このコメントは、関数名と同じ情報しか持たない。処理列で示したいのは「この関数を呼ぶ」ではなく、「一連の利用目的の中で、ここではJobを完了させる」という段階である。

実装を始める前に処理列を置くと、少なくとも次を検討できる。

- 何をもって成功とするか
- どの順序に意味があるか
- どの段階が別概念の能力を利用するか
- どの段階にFeature固有の判断があるか
- どこまでを一緒に失敗させるか
- 結果を外部へ返す前に何を確定するか

処理列が正解を生成するわけではない。曖昧な考えを、検討できる粒度へ出すことに価値がある。

## 3. コメントの粒度を揃える

次のコメントは、一行の中に異なる詳細と判断主体を含んでいる。

```python
# DatabaseからJobを取得し、状態を検証して更新し、
# 権限を考慮して通知先を選択する
```

ここには、永続化、Job状態の規則、権限、通知先の選択が混在している。処理列の一行としては、UseCaseが何を利用するのか、内部で何を判断するのかが分かれない。

例えば次のように、UseCaseから見た処理段階へ戻せる。

```python
# 操作者がJobを完了できることを確認する

# Jobを完了させる

# 完了通知の依頼を記録する
```

一行に動詞が複数あること自体は問題ではない。「Jobを取得して完了させる」がJob Data Componentの一つの公開操作として成立する場合もある。

確認するのは、同じ抽象度と判断主体を保っているかである。

- 「Jobを完了させる」は、Jobの能力を利用する段階として読める
- 「Runningだけ完了可能なので、statusを調べる」は、Job内部の規則をFeature側が説明し始めている
- 「権限を考慮して通知先を選ぶ」は、権限判断と通知先選択の主体が混ざっている
- 「通知の依頼を記録する」は、実際の配送方法を知らずに利用できる能力として読める

「そして」「条件に応じて」「ただし」が増えたことは観測点になるが、機械的な分割条件にはしない。何についての判断が増え、誰がその判断を持つべきかを見る。

## 4. 処理列から直接利用を取り出す

処理列を実装すると、Complete Jobが直接利用する能力が現れる。

```python
# feature/complete_job.py

from dataclasses import dataclass
from typing import Self


@dataclass(frozen=True, slots=True)
class Dependencies:
    ensure_permission: EnsurePermission
    complete_job: CompleteJob
    enqueue_completion_notice: EnqueueCompletionNotice

    @classmethod
    def default(cls) -> Self:
        from data_component.job import complete_job
        from permission import ensure_job_completion
        from notification import enqueue_completion_notice

        return cls(
            ensure_permission=ensure_job_completion,
            complete_job=complete_job,
            enqueue_completion_notice=enqueue_completion_notice,
        )


def execute(
    command: CompleteJobCommand,
    connection: Connection,
    *,
    dependencies: Dependencies | None = None,
) -> None:
    if dependencies is None:
        dependencies = Dependencies.default()

    # 操作者がJobを完了できることを確認する
    dependencies.ensure_permission(
        command.actor_id,
        command.job_id,
        connection,
    )

    # Jobを完了させる
    dependencies.complete_job(
        command.job_id,
        connection,
    )

    # 完了通知の依頼を記録する
    dependencies.enqueue_completion_notice(
        command.job_id,
        connection,
    )

    # 一連の変更を確定する
    connection.commit()
```

このModuleが直接利用するのは、権限確認、Jobの公開操作、通知依頼の記録、Connectionである。Job Repositoryや通知配送Clientのように、呼び出した先のModuleだけが利用する依存まで、Featureの`Dependencies`へ輸送しない。

`Dependencies.default()`は、通常実行で使う標準構成を選ぶ一案である。同時に、Module Testでは直接依存を明示的に差し替えられる。

```python
dependencies = Dependencies(
    ensure_permission=fake_ensure_permission,
    complete_job=fake_complete_job,
    enqueue_completion_notice=fake_enqueue_notice,
)

execute(
    command,
    fake_connection,
    dependencies=dependencies,
)
```

この段階で確保しているのは、処理本体が直接依存を受け取れることと、標準構成とは別にTestできることである。

一方、`default()`が正しいProduction実装を選ぶこと、Database Poolを一度だけ生成すること、Process停止時にResourceを破棄することまでは保証しない。標準構成の解決と長寿命Resourceのlifecycleは、第7章と第14章で分けて扱う。

## 5. 利用と判断を分ける

Featureが別Moduleを呼ぶことは、責任を失ったことを意味しない。Complete Jobは、どの能力をどの順序で利用し、どこまでを一つの成功として扱うかを決める。

一方、利用先の内部判断までFeatureが持ち始めたなら、判断主体が混ざっていないかを見る。

### Featureへ残せる判断

- Complete Jobでは完了理由を必須にする
- 一連の変更と通知依頼の記録を同じTransactionで成功させる
- Previewの場合は状態を変更せず結果だけ返す
- 操作者へ返す完了結果に何を含めるか決める

### 別概念の候補になる判断

- Jobのどの利用でも守る状態遷移規則
- Permissionのどの利用でも共通するRole判定
- 通知provider固有のRetry可能なError判定
- Database RowからJob状態を復元する規則

例えば、Complete Jobでは完了理由を必須にするとする。

```python
if not command.reason.strip():
    raise ValueError("completion reason is required")
```

この条件がComplete Jobという利用目的だけに属するなら、Featureへ置ける。

一方、Completed Jobは再度完了できないという条件が、どのFeatureから利用してもJobに共通するなら、Job Data Componentへ置く候補になる。

Domain LogicかApplication Logicかという分類名だけで配置を決めず、何についての判断か、どの利用で同じ規則として守るかを見る。

## 6. 完了の意味からTransaction候補を出す

処理列の最後に「一連の変更を確定する」と書くと、どこまでを一緒に成功させたいかを検討できる。

この例では、Jobの状態変更と通知依頼の記録を同じDatabaseへ保存している。両方が揃って初めてComplete Jobが成功すると判断するなら、同じTransactionに含められる。

実際のMail送信や外部API呼び出しは、Databaseのrollbackで取り消せない。次の二つは同じ意味ではない。

```text
通知の依頼をDatabaseへ記録する
実際に外部へ通知を配送する
```

外部配送までを一つのDatabase Transactionに見せると、commit後の送信失敗、重複送信、再試行、Timeoutを別途扱う必要がある。処理列は、技術的に原子的でない処理を原子的に見せるためではなく、その差を表へ出すために使う。

Transaction境界の実装は第8章で扱う。ここでは、Featureが利用目的の成功範囲を知る場所であることだけを先に示す。

## 7. 分岐は意味の差から考える

条件分岐があるだけでは、Featureを分けない。

```python
if command.preview:
    return build_preview(command)

apply_completion(command, connection)
connection.commit()
```

分岐について、少なくとも次を確認する。

- 利用者は同じか
- 必要な権限は同じか
- 成功条件は同じか
- 副作用は同じか
- 返す結果は同じ目的に属するか
- 一方だけを独立して変更、公開、Testする必要があるか

Previewと本実行が、同じ画面操作の一部として常に一緒に変わるなら、一つのFeature内部に残せる。

Previewだけが別の利用者へ公開され、権限や結果契約も独立するなら、`preview_job_completion.py`という兄弟Featureの候補になる。

同じ能力についてProduction実装とTest実装を選ぶだけなら、利用目的の分岐ではなくDependency選択として扱える。

分岐数ではなく、分岐によって何の意味が変わるかを見る。

## 8. Loopでは集合単位と一件単位を分ける

複数Jobを完了するBatch Featureでは、処理列を集合と一件の二つの粒度で考える。

```python
def execute(...):
    # 処理対象のJobを選ぶ

    # 各Jobへ同じ完了処理を適用する

    # 結果を集計する

    # 一連の変更を確定する
```

Loopは多くの処理を一行へ圧縮するため、その内部を観測する。

- 各件が独立して成功・失敗するなら、一件用の公開操作をLoopから利用できる
- 全件成功が必要なら、Batch FeatureがTransaction境界を持つ候補になる
- 項目間の順序や整合性があるなら、集合自体が独立した概念候補になる
- 一件ごとの失敗処理が複雑なら、Loop内部の処理へ名前を与える候補になる
- 一件用UseCaseをそのまま呼ぶと内側でcommitするなら、Transaction境界を再設計する必要がある

「既存UseCaseを再利用できるから」という理由だけで、UseCaseを入れ子にしない。処理本体の公開操作を合成するのか、より大きな利用目的へTransactionを持ち上げるのかを検討する。

## 9. Entry固有のUseCaseを許容する

UseCaseという役割を持つコードを、必ず`feature/`へ置く必要はない。

Web Sessionの復旧にしか意味を持たない処理なら、次の配置が成立する。

```text
entry/
    web/
        restore_session.py
```

この処理は一連の利用目的を持つが、Web Sessionという親概念がなければ同じ意味では成立しない。

後からCLIや別Protocolでも使える独立した利用目的だと分かったなら、Featureへの移動を検討する。利用箇所が二つになったから移すのではなく、Webという親概念なしでも同じ意味で成立すると判断したため移す。

## 10. ModuleからPackageへ成長させる

Complete Jobが一つのModuleで読める間は、`feature/complete_job.py`でよい。

```text
feature/
    complete_job.py
```

Feature固有のPreview処理が増え、UseCase本体と別に変更・Testする理由が現れたなら、同名Packageへ成長させられる。

```text
feature/
    complete_job/
        __init__.py
        usecase.py
        preview.py
```

`__init__.py`は通常利用する入口を示す。

```python
from .usecase import CompleteJobCommand, execute

__all__ = ["CompleteJobCommand", "execute"]
```

内部ファイルを増やす前に、次を確認する。

- 同じComplete Jobという親概念で説明できるか
- 独立した役割と変更理由があるか
- 別概念から通常利用させるものは何か
- Package内部だけで使うものは何か
- 兄弟Featureへ分ける方が目的を説明しやすくないか

概念を一つ作ることと、そのために複数ファイルを作ることは別である。

## 11. 処理列だけでは表せないものを認める

処理列は、直線的なUseCaseを整理するのに向いている。一方、次の処理は、コメントを上から並べるだけでは十分に表現できない。

- 非同期Eventが連鎖する処理
- 複数Transactionと待機をまたぐWorkflow
- 並行処理
- 再帰的な処理
- 状態遷移そのものが中心となる処理
- 補償処理を持つ長期Process

その場合は、状態図、遷移表、Event一覧、時系列、Workflowなど、対象に合う表現を追加する。

処理列は、大きなUseCaseをそのまま正当化する道具でもない。コメントと実装の対応が崩れたら、処理単位、関数、公開面、配置を見直す。

## 12. この章の結論

Featureは利用目的に名前を与え、UseCaseはその完了までの処理順序を表す。

複数行の実装を一つの処理段階としてコメントへ置くと、直接利用、判断主体、成功範囲、分岐、Loopの粒度を検討できる。コメントそのものを完成条件にはせず、得られた判断をModule、Package、公開面、Dependencies、Transactionへ反映する。

次章では、複数Featureから観測したJobの状態、規則、同一性、永続化をData Componentへ置き、Applicationをさらに成長させる。

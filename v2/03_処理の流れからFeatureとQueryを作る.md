# 第3章　処理の流れからFeatureとQueryを作る

## この章の目的

利用目的が完了するまでの処理を同じ抽象度で並べ、Feature、UseCase、直接利用する概念、分岐、繰り返しを具体化する。あわせて、読み取り処理であるQueryを利用目的へ配置する。

## 1. Featureは利用目的に名前を与える

Featureは、利用者またはシステムが達成しようとする目的をまとめる。

```text
feature/
    create_job.py
    complete_job.py
    reschedule_job.py
```

FeatureはApplication層の別名ではない。`complete_job`という名前から、そのコードが何のために存在するかを説明できることが重要である。

### Featureを最終単位に固定しない

利用目的に必要な処理をFeatureの近くへ置くと、結果として一つの利用経路が縦にまとまった構造になることがある。ただし、Featureを分割の終端にはしない。

複数Featureに同じ状態と規則が現れればData Componentを検討する。Entryがなければ成立しない処理はEntry配下に置ける。外部サービスとの通信が独立した意味を持てば、別の兄弟概念を作れる。

一時点の構造がVertical Sliceに似ていても、その形を完成条件にはしない。増えた具体例によって、横断する概念と親子関係を更新する。

## 2. 最初に完了までの処理列を書く

UseCaseを実装する前に、一連の処理が何をもって完了するかをコメントで並べられる。

```python
def execute(...):
    # 操作者がJobを完了できることを確認する

    # Jobを完了させる

    # 完了を関係者へ通知する

    # 一連の変更を確定する
```

コメント一行を、UseCaseから見た一つの処理単位にする。

これは、次のようにコード一行を日本語へ置き換えるコメントとは異なる。

```python
# complete_job関数を呼ぶ
complete_job(job_id, connection)
```

処理列のコメントは、複数行の実装を一つの目的としてまとめる。実装方法が変わっても、その段階の意味が変わらなければコメントは残る。

処理列は、配置候補や直接利用を検討するための補助手段である。コメント形式そのものを中核原則にはしない。判断結果は、最終的にファイル、フォルダ、公開面、依存関係へ反映する。

## 3. 処理列から直接利用を取り出す

処理列を実装すると、UseCaseが直接利用する能力が現れる。

```python
def execute(command, *, dependencies):
    # 操作者がJobを完了できることを確認する
    dependencies.ensure_permission(
        command.actor_id,
        command.job_id,
    )

    # Jobを完了させる
    dependencies.complete_job(
        command.job_id,
        dependencies.connection,
    )

    # 完了を関係者へ通知する
    dependencies.send_notification(command.job_id)

    # 一連の変更を確定する
    dependencies.connection.commit()
```

ここから、権限確認、Jobの公開操作、通知、ConnectionがUseCaseの直接利用だと分かる。

処理列は依存を自動的に決めるものではない。どの処理をFeature自身が判断し、どの処理を別概念へ委ねるかを検討するための材料になる。

## 4. 同じ抽象度を保つ

次のコメントは、一行の中に異なる詳細を含んでいる。

```python
# DatabaseからJobを取得し、状態を検証して更新し、
# 権限を考慮して通知先を選択する
```

この記述には、永続化、状態規則、権限、通知先選択が混在している。

すぐに四つのModuleへ分割する必要はない。しかし、現在のUseCaseが下位概念の詳細まで抱えていないか、処理単位を分けられないかを検討する観測点になる。

「そして」「条件に応じて」「ただし」が一行へ増えたことも観測点にはなるが、機械的な分割条件ではない。

## 5. Feature固有の判断とData Componentの規則を分ける

Complete Jobでは完了理由の入力を必須にするとする。

```python
if not command.reason.strip():
    raise ValueError("completion reason is required")
```

この規則がComplete Jobという利用目的だけに属するなら、Featureへ置ける。

一方、Completed Jobは再度完了できないという規則が、どのFeatureから利用してもJobに共通するなら、Job Data Componentの候補になる。

Domain LogicかApplication Logicかという分類名だけで決めず、何についての判断かを見る。

## 6. 分岐は完了の意味から考える

条件分岐があるだけではFeatureを分けない。

```python
if command.preview:
    return build_preview(...)

apply_change(...)
connection.commit()
```

Previewと本実行で、利用者、成功条件、副作用、権限が大きく異なるなら、別Featureの候補になる。

同じ能力について実装だけを選ぶ分岐なら、依存選択や外部サービス側へ置く候補になる。

一つの利用目的の中で明示すべき代替経路なら、UseCaseに残せる。

分岐数ではなく、分岐によって何の意味が変わるかを見る。

## 7. Loopでは集合単位と一件単位を分ける

Batch処理では、処理列を二つの粒度で考える。

```python
# 処理対象のJobを選ぶ

# 各Jobへ同じ完了処理を適用する

# 結果を集計する

# 一連の変更を確定する
```

- 各件が独立しているなら、一件用の公開操作をLoopから利用できる
- 全件成功が必要なら、Batch FeatureがTransaction境界を持つ候補になる
- 項目間の順序や整合性があるなら、集合自体が概念候補になる
- 一件ごとの失敗処理が複雑なら、Loop内部の処理へ名前を与える候補になる

Loopは複数の処理を一行へ圧縮するため、内部の粒度を意識して観測する。

## 8. Queryも利用目的へ置く

Queryは、状態を変更せず必要な情報を読む役割である。Queryという役割だけを理由に、トップレベルの`query/`を作る必要はない。

Complete Jobの確認画面だけに必要なら、次のように置ける。

```text
feature/
    complete_job/
        usecase.py
        query.py
```

一覧表示そのものが利用目的なら、読み取りだけのFeatureを作れる。

```text
feature/
    list_jobs.py
```

```python
@dataclass(frozen=True, slots=True)
class JobListItem:
    job_id: int
    name: str
    status: str


def execute(condition, connection) -> list[JobListItem]:
    rows = connection.execute(...)
    return [JobListItem(...) for row in rows]
```

読み取りのためだけに、完全なDomain Modelを復元する必要はない。利用目的が必要とするRead Modelを直接作ってよい。

同じSQLが二箇所に現れたことだけで共通化しない。同じ読み取り概念として変更されるかを見る。

## 9. Entry固有のUseCaseを許容する

UseCaseという役割だから必ず`feature/`へ置くわけではない。

Web Sessionの復旧にしか意味を持たない処理なら、次の配置が成立する。

```text
entry/
    web/
        restore_session.py
```

後からWebと無関係な意味を持ったなら、Featureへの移動を検討する。

## 10. 処理列の限界

処理列は、直線的な業務処理を整理するために有効である。一方、次の処理では、コメントを並べるだけでは構造を十分に表現できない。

- 非同期Eventが連鎖する処理
- 長時間継続するWorkflow
- 並行処理
- 再帰的な処理
- 状態遷移そのものが中心となる処理

その場合は、状態図、Event、Workflow、遷移表など、対象に合う表現を追加する。

コメントは大きなUseCaseを正当化する道具でもない。コメントと実装の対応が崩れたら、名前、関数、公開面、配置を見直す。

## 11. この章の結論

Featureは利用目的に名前を与え、UseCaseはその完了までの処理順序を表す。

処理列のコメントは、コード一行の説明ではなく、UseCaseから見た処理単位を先に置く方法である。処理列から直接利用、判断主体、分岐、Loopの粒度を観測できる。ただし、処理列だけで概念境界が決まるわけではなく、状態やライフサイクルの観測と組み合わせる。

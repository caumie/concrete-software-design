# 第3章　`app.py`から三つの初期構造へ成長させる

## この章の目的

小さなApplicationを`app.py`一つから書き始め、実装によって区別できる意味が増えた時点で、`data_component / feature / entry`へ成長させる。

本章では、最終的なフォルダtreeを先に提示して、その形へコードを当てはめない。要求を一つずつ追加し、その時点で観測できる事実、配置候補、暫定判断、再検討する条件を順に示す。

この事例はReference Architectureではない。別のApplicationでも同じ順番、同じ名前、同じファイル数になるとは限らない。示したいのは、`app.py`から始めても設計を放棄したことにはならず、意味の差が現れてから構造を増やせるという判断過程である。

## 1. Job登録だけを一つのModuleへ置く

最初の要求は、Jobを登録することだけとする。永続化も、まずはProcess内のListでよい。

```text
app.py
```

```python
# app.py

jobs: list[dict] = []


def create_job(name: str) -> dict:
    job = {
        "id": len(jobs) + 1,
        "name": name,
        "status": "created",
    }
    jobs.append(job)
    return job
```

### 観測した事実

- 利用目的はJob登録一つだけである
- 入力方法は関数引数だけである
- 状態規則、永続化、外部表現を別の意味として説明する材料が少ない
- 一つの関数と一つのListで処理全体を追える

### 配置候補

1. `app.py`へ置く
2. 将来を見越して`feature/create_job.py`を作る
3. `domain/`、`repository/`、`service/`を先に作る
4. `data_component/`、`feature/`、`entry/`を空のまま用意する

### 暫定判断

`app.py`へ置く。

この段階で複数の役割名を導入しても、それぞれへ置く具体的な差がない。将来Databaseを使いそうであることは、現在Repositoryを必要とする理由にはならない。将来Webから呼ばれそうであることも、空の`entry/`を作る理由にはならない。

一つのModuleに置くことは、Application全体を今後も一つのファイルへ閉じ込める宣言ではない。現在区別できる意味が一つなので、構造も一つに留めるという判断である。

### 再検討する条件

- HTTP、CLI、Batchなど外部経路固有の処理が加わる
- Job登録とは異なる利用目的が加わる
- 複数の処理で同じJob状態や規則を扱う
- 永続化の取得、復元、保存を独立した役割として説明できる
- `app.py`の一部を変える理由が、他の部分を変える理由と明確に分かれる

再検討条件は、行数の上限ではない。何の意味が増えたら現在の配置理由が崩れるかを残す。

## 2. Webという外部経路を分ける

次に、Web RequestからJobを登録し、Databaseへ保存する要求が加わったとする。

```python
def create_job_handler(request):
    name = request.form["name"]
    connection = connect_database()
    connection.execute(
        "INSERT INTO job (name, status) VALUES (?, ?)",
        (name, "created"),
    )
    connection.commit()
    return render_template("created.html")
```

一つの関数に、少なくとも次の内容が現れた。

- Formから値を読む
- Jobを登録する
- Databaseへ書き込む
- Transactionを確定する
- HTML Responseを作る

ここで動詞が複数あることだけを理由に、すべてを別Moduleへ分けるわけではない。まず、どの意味の差を説明できるかを見る。

### 観測した事実

- Request、Form、Template、ResponseはWebという経路がなければ成立しない
- Job登録はWeb以外から呼ばれても同じ利用目的として成立し得る
- Databaseは導入されたが、取得・復元・保存の複数操作や、独立したJob永続化の変更理由はまだ観測していない

### 配置候補

1. すべて`app.py`へ残す
2. Web固有処理だけをEntryとして分ける
3. Web、Feature、Data Component、Repositoryを一度に作る

### 暫定判断

Web固有処理を`entry/web.py`へ分ける。

```text
entry/
    web.py

app.py
```

```python
# app.py

def create_job(name: str, connection) -> int:
    cursor = connection.execute(
        "INSERT INTO job (name, status) VALUES (?, ?)",
        (name, "created"),
    )
    return int(cursor.lastrowid)
```

```python
# entry/web.py

from app import create_job


def create_job_handler(request):
    name = request.form["name"]

    with request.app.state.database.connect() as connection:
        job_id = create_job(name, connection)
        connection.commit()

    return render_template("created.html", job_id=job_id)
```

この時点では、`app.py`に置かれた処理を完成形とはみなさない。Webの都合だけを先に分け、Job登録の利用目的と永続化をさらに分けるかは、次の具体例を待つ。

`entry/`を作ったからといって、残り二つの親フォルダも揃える必要はない。三つの初期構造は、常に同時に作成するTemplateではない。

## 3. 異なる利用目的が現れてからFeatureを作る

Job登録に加えて、Jobを完了する要求が加わったとする。

完了処理を実装する前に、何をもって一連の処理が完了するかを並べる。

```python
def complete_job(...):
    # 操作者がJobを完了できることを確認する

    # Jobを完了させる

    # 完了通知の依頼を記録する

    # 一連の変更を確定する
```

このコメントはコード一行ずつを日本語に置き換えたものではない。各行は、複数行の実装を含み得る一つの処理段階である。処理方法が変わっても「Jobを完了させる」という段階が残るなら、コメントの意味も残る。

### 観測した事実

- Create JobとComplete Jobは、異なる入力と成功条件を持つ
- Complete Jobには権限確認、状態変更、通知依頼、Transactionという固有の処理列がある
- Web以外のEntryからも同じ目的を実行し得る
- Job状態の規則が一つの概念へ集まるかは、まだ十分に観測できていない

### 配置候補

1. `app.py`へ二つの関数を並べる
2. 利用目的ごとにFeatureを作る
3. Jobという名前の一つの大きなModuleへ、すべての操作を集める

### 暫定判断

Create JobとComplete Jobを、兄弟のFeatureとして置く。

```text
feature/
    create_job.py
    complete_job.py

entry/
    web.py

app.py
```

`feature/`はApplication層の言い換えではない。`create_job`と`complete_job`という名前によって、誰かまたはシステムが何を達成しようとしているかを配置へ表す。

一つのFeatureは、まず一つのModuleで始める。

```python
# feature/complete_job.py

def execute(command, connection, *, dependencies):
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

ここでは、Complete Jobが何を順番に利用するかをFeatureへ置く。Jobの内部状態、権限の保存形式、通知配送の技術詳細までFeatureが説明する必要はない。

Featureを作った時点で、`command.py`、`handler.py`、`service.py`、`dto.py`などを一式作らない。内部役割を分ける具体的な理由が現れるまでは、`complete_job.py`一つでよい。

## 4. 複数Featureに同じ状態と規則が現れる

その後、Change Job StatusとCancel Jobが加わったとする。実装すると、次の規則が複数Featureへ現れた。

- Completed Jobは変更できない
- Running Jobだけを完了できる
- Cancelled Jobは再開できない
- どの処理もJob IDによって同じJob状態を取得し、変更し、保存する

### 観測した事実

- 規則はComplete Jobだけの入力条件ではなく、Job状態のどの利用でも守る必要がある
- Jobには識別子がある
- JobにはCreated、Running、Completed、Cancelledへ変化するライフサイクルがある
- 複数のFeatureが同じ状態の意味を利用している

### 配置候補

1. 規則を各Featureへ残す
2. `shared/job_rules.py`へ置く
3. `data_component/job.py`を作る

各Featureへ残せば、利用目的ごとの変更は局所化できる。しかし、同じJob状態の意味が重複する。

`shared/job_rules.py`へ置けば重複は減る。しかし、なぜそれらの規則が一緒に置かれるかを「共有されるから」以上に説明しにくい。

`data_component/job.py`へ置けば、Jobの状態、同一性、規則、ライフサイクルを中心に説明できる。

### 暫定判断

Job Data Componentを作る。

```text
data_component/
    job.py

feature/
    create_job.py
    change_job_status.py
    complete_job.py
    cancel_job.py

entry/
    web.py

app.py
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

Featureは、Job状態について共通する規則をJobへ委ねる。一方、Complete Jobで完了理由を必須にするなど、その利用目的だけに属する入力条件はFeatureへ残せる。

Data Componentを作ったことは、正しいDomainを発見した証明ではない。現在観測できたJobの意味を、一つの配置仮説として置いたにすぎない。

## 5. 三つの初期構造が同時に見える

ここまで進んで、初めて三つの親概念が実際のコードを伴って揃った。

| 配置 | この事例で置く意味 | 親概念がなければ成立するか |
| --- | --- | --- |
| `data_component/job.py` | Jobの状態、同一性、共通規則 | Jobという意味が必要 |
| `feature/complete_job.py` | Jobを完了する利用目的と処理順序 | Webがなくても成立する |
| `entry/web.py` | Request、Form、Response、Route | Webがなければ成立しない |

三つのフォルダは役割名を均等に配る箱ではない。

- `data_component/`には、何を保持・参照・変更するかという意味を置く
- `feature/`には、何を達成するかという意味を置く
- `entry/`には、どの外部経路から利用されるかという意味を置く

同じ処理を三層へ分割することが目的ではない。例えば、HTTPの入力検証はWeb Entryへ置ける。Complete Jobという利用目的だけに必要な完了理由の検証はFeatureへ置ける。Completed Jobを再変更できないという規則はJob Data Componentへ置ける。同じ「検証」でも、何についての判断かによって配置が変わる。

## 6. 配下、利用、実行順序を混同しない

この構造から、単純な三層Architectureを想定する必要はない。

```text
entry/web.py
    ↓ 利用
feature/complete_job.py
    ↓ 利用
data_component/job.py
```

この利用関係は典型例にはなるが、フォルダの上下だけで強制される規則ではない。

- `entry/web.py`の配下にあるコードは、Webという親概念で説明できる
- `feature/complete_job.py`がJobを利用しても、JobがComplete Jobの配下になるわけではない
- `data_component/job.py`が物理的に下位層であると決まるわけではない
- Web Sessionの復旧のようにEntryがなければ成立しないUseCaseは、`entry/web/`へ置ける
- 新しい兄弟概念が必要なら、三分類の外へ追加できる

物理的な配下、importによる利用、実行時の呼び出し順序は別の関係である。treeだけで依存関係を推測せず、公開面と直接importも確認する。

## 7. `app.py`を空にすることは目的ではない

FeatureとEntryを分けた後も、`app.py`にはApplication起動時の責任を置ける。

```python
# app.py

from entry.web import create_web_app


def main() -> None:
    application = create_web_app()
    application.run()


if __name__ == "__main__":
    main()
```

Database Poolや長寿命ClientをProcess起動時に作るなら、そのlifecycleを`app.py`または具体的なEntryの起動Moduleへ置くこともできる。何も置かれていないRoot Moduleにするために、起動責任まで遠くへ移す必要はない。

反対に、`app.py`へ業務処理が残っていること自体を直ちに失敗とみなさない。残った処理に独立した意味が観測できるか、現在の親概念で説明できるかを確認する。

## 8. 横への追加と縦への深掘りを分ける

新しい利用目的が増えたなら、Featureを横へ追加できる。

```text
feature/
    create_job.py
    complete_job.py
    reschedule_job.py
```

一つのFeatureの内部に複数の役割が現れたなら、同名Packageへ成長させられる。

```text
feature/
    complete_job/
        __init__.py
        usecase.py
        preview.py
```

ただし、Previewが本実行とは異なる利用者、権限、副作用、成功条件を持つなら、`preview_job_completion.py`という兄弟Featureにする候補もある。ファイル数ではなく、同じ利用目的の内部か、別の利用目的かで判断する。

Data ComponentやEntryも同じように成長できる。

```text
data_component/
    job.py
    schedule.py

entry/
    web.py
    cli.py
```

横への追加は、兄弟となる意味が増えたことを表す。縦への深掘りは、一つの意味の内部役割が増えたことを表す。両者を同じ「ファイルが増えた」という理由だけで選ばない。

## 9. 初期構造は変更費用を消さない

`data_component / feature / entry`は、後から分割、統合、移動、改名できる。しかし、変更可能であることと、無料で変更できることは別である。

構造を変えれば、次も変わり得る。

- import path
- 公開面
- TestのPatch先とFixture
- Dependencyの定義場所
- Transactionの受け渡し
- 開発文書と運用手順
- 他Moduleや外部利用者との互換性

初期構造が提供するのは、変更を局所化し、判断を遅らせられる余地である。適切な境界を自動的に選ぶことでも、移行の痛みをなくすことでもない。

そのため、構造を作るときには、選んだ理由だけでなく、どの事実が増えたら見直すかも残す。

## 10. この事例が示すことと示さないこと

この事例が示すのは、次の流れである。

1. 利用目的が一つなら`app.py`一つで始める
2. Web固有の意味が現れたらEntryを分ける
3. 異なる利用目的が現れたらFeatureを並べる
4. 複数Featureに共通する状態、規則、同一性が現れたらData Componentを検討する
5. 観測した意味だけを配置し、内部役割を先に一式作らない
6. 配置ごとに再検討条件を残す

この事例だけでは、すべてのApplicationが`app.py`から始まることも、三つのフォルダが必ず必要になることも、同じ段階で同じ境界が現れることも示さない。

重要なのは、最終treeを模倣することではない。現在のコードで区別できる意味だけを置き、実装によって差が増えたときに構造へ反映することである。

## 11. この章の結論

小さなApplicationは、`app.py`一つから始められる。外部経路、利用目的、意味ある状態という差が観測できたときに、`entry / feature / data_component`を順に作ればよい。

三つの初期構造は、最初に完成形を決めるTemplateではない。実装から得た事実へ名前を与え、配置として残すための初期値である。次章では、この流れの中で作ったFeatureを、処理列、直接依存、分岐、Loopの単位まで具体化する。

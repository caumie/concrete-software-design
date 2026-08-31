# 第10章　Entryを具体化する

## 1. Entryは外部からシステムへ入る経路を扱う

具象設計論では、外部からシステムへ処理が入ってくる経路を **Entry** と呼ぶ。

最初は、

```text
entry/
    web.py
```

のように一つのModuleでよい。

CLIも必要になれば、

```text
entry/
    web.py
    cli.py
```

と兄弟として追加できる。

Webが大きくなれば、

```text
entry/
    web/
        __init__.py
        routes.py
        session.py
```

へPackage化できる。

EntryはInbound側の概念である。

- HTTP Request
- CLI argument
- Batch起動
- Message consumer

など、外部からシステムを利用する経路を扱う。

一方、システムから外部APIを呼ぶ処理はEntryではない。External Service等のOutbound側の概念として別に扱う。

## 2. WebとCLIはEntryの具体例として兄弟にできる

最初はWebしか存在しなかったとする。

```text
entry/
    web.py
```

後からCLIを追加する。

```text
entry/
    web.py
    cli.py
```

とすればよい。

Webの内部へCLIを置く必要はない。

さらにWebだけが複雑になれば、

```text
entry/
    web/
        __init__.py
        routes.py
        session.py
    cli.py
```

と縦に深掘りできる。

`entry`という抽象的な親概念を置くことで、Webという具体技術を最初のトップレベル構造として固定せずに済む。

## 3. Requestを内部で使う値へ変換する

Web Entry Handlerでは、外部表現をFeatureが利用できる値へ変換する。

```python
from feature.complete_job import execute


def handle(
    request,
) -> Response:
    job_id = int(
        request.path_params["job_id"]
    )
    reason = request.form["reason"]

    execute(
        job_id=job_id,
        reason=reason,
    )

    return redirect(...)
```

HTTP RequestそのものをUseCaseへ渡す必要はない。

Featureが必要なのは、

- `job_id`
- `reason`

である。

Entryは外部表現を内部呼び出しへ具体化する。

---

## 4. 構文上のValidationは入口に置ける

例えばHTTP Requestに、

```text
job_id=abc
```

が来た。

整数として解釈できないことは、Web入力の問題として扱える。

```python
try:
    job_id = int(
        request.path_params["job_id"]
    )
except ValueError:
    return bad_request(...)
```

一方、

> Completed Jobは変更できない

という規則はWebの入力形式ではない。

Job Data Component側の意味になる。

同じ「Validation」という名前でも、変更理由が異なる。

具象設計ではValidationを一つの共通層へ集約しない。

---

## 5. Responseへの変換も入口の責任である

Featureが、

```python
CompleteJobResult(
    job_id=10,
    completed_at=...
)
```

を返すとする。

Web側で、

```python
return JSONResponse(
    {
        "job_id": result.job_id,
        "completed_at": result.completed_at.isoformat(),
    }
)
```

と変換できる。

CLIなら、

```python
print(
    f"completed: {result.job_id}"
)
```

とするかもしれない。

FeatureがJSONやExit Codeを知る必要はない。

---

## 6. Framework固有型を必要以上に内側へ運ばない

FastAPI、Django、FlaskなどのRequest ModelやResponse Modelは、Webという概念に属する。

例えば、

```python
class CompleteJobRequest(BaseModel):
    reason: str
```

をそのままData Componentへ渡す必要はない。

UseCaseに必要な値へ変換する。

ただし、Framework型を一切使ってはならないという規則にはしない。

`entry/web`配下の処理にしか意味を持たないUseCaseなら、Framework型を使う合理的な理由もあり得る。

目的はFramework依存をゼロにすることではない。

**Frameworkの都合が無関係な概念まで広がらないようにすること**である。

---

## 7. 入口を薄くすることを目的にしない

よくある設計規則として、

> Controllerは薄くする

がある。

これは有用な目安だが、薄さ自体を目的にすると、入口固有の判断まで別層へ押し出してしまう。

例えば、

- HTTP Headerを読む
- Cookieを更新する
- Pagination parameterを解釈する
- Templateを選ぶ
- Web固有のErrorをResponseへ変換する

といった処理は、Webに置いてよい。

Entryは薄い必要はない。

**外部表現に由来する責任が、外部表現の概念内に閉じていること**が重要である。

---

## 8. Entry固有UseCase

Webでしか成立しない処理もある。

例えばSession復旧を考える。

```text
entry/
    web/
    restore_session/
        usecase.py
```

このUseCaseが、

- Cookie
- Web Session
- Browser redirect

に強く依存し、Web以外では意味を持たないなら、`entry/web`配下に置いてよい。

UseCaseという役割だからFeatureへ移す必要はない。

---

## 9. Entryの意味を越えたらFeatureへ移す

最初はWeb固有だと思っていた処理が、後からCLIでも使われるようになったとする。

```text
entry/
    web/
    import_job/
        usecase.py
```

が、

```text
cli/
```

からも必要になり、HTTPやWeb Sessionとは無関係な処理だと分かった。

この時点で、

```text
feature/
    import_job/
```

へ移すことを検討できる。

利用箇所が二つになったからではない。

Webという親概念がなくても同じ意味で成立するようになったからである。

---

## 10. EntryにもDependencyを局所的に持てる

Web Entry Handlerが、

- Session Provider
- Feature公開関数

を直接利用するなら、そのモジュールで依存を定義できる。

```python
@dataclass(frozen=True, slots=True)
class Dependencies:
    complete_job: CompleteJob
    session: SessionProvider

    @classmethod
    def default(cls) -> Self:
        return cls(
            complete_job=complete_job_default,
            session=session_provider,
        )
```

Feature内部のDependenciesを入れ子にしない。

Webが直接使うものだけをWebのモジュールで持つ。

---

## 11. EntryでConnectionを作るか、UseCaseに持たせるか

UseCase自身の`Dependencies.default()`がConnectionを生成する設計なら、Web EntryはConnectionを知らずにFeatureを呼べる。

```python
execute(
    job_id=job_id,
    reason=reason,
)
```

一方、Request単位でConnection lifecycleをWeb Frameworkが管理する設計もあり得る。

その場合は、

```python
execute(
    ...,
    dependencies=...
)
```

など、Frameworkとの接続方法を調整する。

具象設計論が固定するのは特定Frameworkとの接続方法ではない。

Transactionの意味をUseCaseが失わないこと、依存が無関係な概念へ伝播しないことを重視する。

---

## 12. MVCの語彙との関係

本書で`web`という名前を使っていても、MVCを否定しているわけではない。

プロジェクトで、

```text
model/
controller/
view/
```

という名前を明確に定義して使うなら、それでもよい。

一方、Webという入口の具体性をそのまま名前にしたいなら`web/`とする。

重要なのは、

> 一般的なArchitecture用語を使ったか

ではなく、

> その名前の配下に何が入るかを説明できるか

である。

---

## 13. Entryの公開面

外部から直接呼び出すEntry Pointがあるなら、Web ApplicationのBootstrapやRouterから利用する公開面を持てる。

ただし、Web内部のすべてを一つの巨大Facadeへ集約する必要はない。

Route単位、画面単位、機能単位など、そのWebプロジェクトで説明できる概念に応じて整理する。

---

## 14. 入口の判断原則

Entryについて次のように考える。

1. 外部表現に由来する責任をEntryへ置く
2. Requestを内部で必要な値へ変換する
3. Responseを外部表現へ変換する
4. 構文Validationと業務規則を分ける
5. Framework型を無関係な概念まで運ばない
6. Entryを薄くすること自体は目的にしない
7. Entry固有UseCaseを許容する
8. Entryの意味を越えた処理は配置を見直す
9. EntryのDependencyもモジュールローカルにする

次章では、通常の関数呼び出しより明確な外部境界が必要になる場合を扱う。

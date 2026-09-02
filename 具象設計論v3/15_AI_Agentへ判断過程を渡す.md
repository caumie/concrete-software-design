# 第15章　AI Agentへ判断過程を渡す

## この章の目的

AI Agentへ完成したフォルダtreeだけを模倣させず、要求から処理列、概念候補、配置、公開面、依存、Testへ進む判断過程を渡す。実装後の俯瞰と、人間が判断すべき構造変更の提示方法を定める。

## 1. 完成形だけを与えない

Agentへ次のtreeだけを与えると、すべてのApplicationへ機械的に再現する可能性がある。

```text
data_component/
feature/
entry/
```

必要なのは、なぜその構造を使い、いつ変更するかという判断過程である。

## 2. 実装前に確認させること

Agentには、コードを追加する前に次を確認させる。

1. 利用者またはシステムが何を完了しようとしているか
2. 完了までの処理列は何か
3. 現在どの概念が存在するか
4. 新しいコードを既存概念の配下で説明できるか
5. 親概念がなくても同じ意味で成立するか
6. どのModuleを直接利用するか
7. 成功・失敗を一緒に扱うTransaction範囲はどこか
8. どのTest境界で確認するか

新しいフォルダを作る前に、独立した意味を説明させる。

## 3. 処理列を先に提出させる

例えばComplete Jobを実装する前に、次の骨格を提示させる。

```text
- 操作者がJobを完了できることを確認する
- Jobを完了させる
- 必要な通知を行う
- 一連の変更を確定する
```

その後、各行について次を示させる。

- Feature自身が判断するか
- 既存Data Componentの公開操作を利用するか
- 外部サービスを利用するか
- 直接依存として何を明示するか

処理列は実装の許可を得るための形式ではない。異なる抽象度や隠れた責任を実装前に確認する材料である。

## 4. 配置判断の途中を示させる

新しい概念を作る場合は、結論だけでなく次を提出させる。

```text
観測した事実:
- Schedule固有の状態変更が3Featureに現れた
- Job完了後もSchedule変更が存在する

候補:
1. Job配下へ残す
2. Job配下でPackage化する
3. Schedule Data Componentへ分ける

暫定選択:
- Schedule Data Componentへ分ける

影響:
- Reschedule JobはJobとScheduleの公開操作を利用する
- 同じConnectionを渡し、Transaction境界は維持する

再検討条件:
- ScheduleがJobと常に同じライフサイクルへ変わった場合
```

Agentが一般的なPattern名を挙げたことを、配置理由として扱わない。

## 5. 通常の配置規則

Agent向けには、次を通常の規則として与えられる。

- 現在説明できる意味を一つのModuleとして始める
- 内部役割が増えたときだけ同名Packageへ成長させる
- 役割名だけを理由にトップレベル層を作らない
- 別概念からは概念トップの公開面を優先して利用する
- Repositoryは対象概念の近くへ置く候補とする
- Queryは何のための読み取りかで配置する
- EntryはInbound経路、外部サービスはOutbound境界として分ける
- 依存は直接利用するModuleで明示する
- 下位Moduleの依存を上位へ輸送しない
- Repositoryはcommitしない
- Transactionは利用目的の成功範囲から決める
- Sharedへの広い依存を機械的に禁止しない

これらは例外を不可能にする規則ではない。異なる配置を選ぶ場合は、その意味と影響範囲を説明させる。

## 6. Python固有規則を中核と分ける

AgentへPython実装規約を与える場合、言語非依存の原則と分けて記述する。

- `Dependencies.default()`を関数のデフォルト引数へ直接置かない
- Module単体Testでは直接依存だけを差し替える
- `__init__.py`でPackageの通常公開面を示す
- `Protocol`は型として契約を表す価値があるときに使う

Local Defaultは自動適用させない。Resource lifecycle、外部Composition、wiring Testの方針がプロジェクトで決まっている場合にだけ使わせる。

## 7. 実装後に俯瞰させる

要求された機能を実装した後、周辺を確認させる。

### 配置

- 新しいファイルを親概念で説明できるか
- 新しいフォルダは独立した意味を持つか
- 名前と配下がずれていないか

### 公開面と依存

- 深いimportが増えていないか
- 直接利用しないDependencyを持っていないか
- 下位ModuleのDependencyを輸送していないか
- 公開面が過度に大きくなっていないか

### 概念

- 同じ状態や規則が複数Featureへ現れていないか
- 独立したライフサイクルが見えていないか
- Sharedに異なる意味が混ざっていないか
- 同時変更の原因が依存先にないか

### Test

- Module Testが直接依存の境界で止まっているか
- 構造変更を支えるTestがあるか

## 8. 大きな構造変更は候補として提示させる

概念の分割、統合、名前変更は影響範囲が大きい。

Agentには、条件を検出しただけで自動Refactoringさせず、次の形で候補を提示させられる。

```text
候補: ScheduleをJob Data Componentから分割する

理由:
- 独立した状態変更が複数Featureに存在する
- Job完了後もScheduleの変更がある

変更範囲:
- Package移動
- 公開面変更
- import更新
- Repository依存の再配置
- Test更新
```

最終判断を人間が行い、採用した場合だけ構造へ反映する運用もできる。

## 9. 正例だけでなく見直し例を与える

Agentは正例の形を表面的に模倣しやすい。配置を見直す例も与える。

```text
見直し前:
feature/complete_job/infrastructure/repository.py

観測:
- RepositoryはComplete Job固有ではなくJobの取得・保存を扱う
- Cancel Jobからも同じJob永続化を利用する

候補:
data_component/job/repository.py
```

または、Dependency階層の見直し例を与える。

```text
見直し前:
CompleteJobDependencies(job=JobDependencies(...))

観測:
- Complete JobがJob Repositoryを直接利用していない

変更:
- Complete JobはJobの公開操作だけを直接依存として持つ
- Job RepositoryはJob Moduleで明示する
```

## 10. Agentへ渡す最短の要約

> 現在説明できる意味を一つのModuleとして置く。役割だけを理由に構造を増やさない。利用目的はFeature、意味あるデータはData Component、外部からの経路はEntryとして始められる。別概念からは公開面を利用し、依存は直接利用するModuleで明示する。実装前に完了までの処理列と配置候補を示し、実装後に増えた具体例を俯瞰する。概念の分割・統合は、観測事実、変更範囲、再検討条件を提示してから行う。

## 11. この章の結論

AI Agentへ渡すべきものは、完成したtreeだけではない。要求から処理列、概念、配置、公開面、依存、Transaction、Testへ進む判断過程である。

正例と見直し例を与え、実装後の俯瞰を要求する。Python固有かつ検証中の構成は中核原則と分け、大きな概念変更は理由と影響を持つ候補として提示させる。

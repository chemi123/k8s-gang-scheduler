# K8s Gang Scheduler 要件定義

## 背景

- 分散学習基盤でVolcanoを使用中。コードが煩雑で用途に対してリッチすぎるため自作する
- K8s v1.36でPodGroup schedulingがalpha導入されているが、インターフェース変更が予想されるため自作しつつ概念レベルで寄せる

## 前提条件

- 外部ログ基盤（Loki, CloudWatch等）が稼働していること。Podは完了/失敗時に即削除されるため、`kubectl logs` でのログ参照は不可

## アーキテクチャ

Controller + Scheduler の2コンポーネント構成。

| コンポーネント | 責務 |
|---|---|
| Controller | GangJob検知 → PodGroup + Pod群(Pending)作成。ライフサイクル管理 |
| Scheduler | Queue内のPodGroupをJob単位で評価し、Pod→Nodeをbind |

kube-scheduler Plugin方式は不採用。理由: Pod単位の処理にJob単位のロジックを押し込む形になり不自然かつ実装が複雑になるため。

独自Schedulerは `schedulerName: gang-scheduler` でkube-schedulerと共存する。

## CRD

| CRD | 作成者 | 役割 |
|-----|--------|------|
| Queue | ユーザー | 研究グループ単位のリソース区画。nodeSelectorでノード群を紐付け |
| GangJob | ユーザー | ジョブ定義（pod template, minNodes, priority, queue） |
| PodGroup | Controller | ランタイム状態（phase, conditions） |

K8s v1.36の Workload(静的テンプレート) / PodGroup(ランタイム) 分離に概念レベルで寄せた設計。

## スケジューリングルール

1. **All-or-nothing** — PodGroupの必要ノード数が揃わなければbindしない
2. **No preemption** — 実行中ジョブを追い出さない。絶対に
3. **Node単位割当** — GPUではなくNode単位でリソース確保。taint `gang.k8s.io/exclusive=true:NoSchedule` でNodeを独占し、kube-schedulerからの配置を防ぐ。操作順序: taint付与 → bind（taint先行でrace windowを防ぐ）。GangJob完了/失敗時にtaintを除去する
4. **同priority内FIFO + head-of-line blocking** — 先頭が詰まったら同priority以下の後続は全て待つ。低priorityジョブのstarvationは許容する（aging等の救済策は設けない）
5. **高priorityのみキュー追い越し可** — 後続jobのpriorityが先頭より厳密に高い場合のみ抜かせる

## Node-Queue関係

- 1 Nodeは複数Queueに所属可能
- Queue CRDがnodeSelectorを持つ（Queue側でNodeを指定する方式）
- Available nodeの計算はQueue問わず「現在使用中でないNode」で判断
- 「使用中」の定義: gang-schedulerがbindしたPodが存在するNode。kube-schedulerが配置した通常Pod、DaemonSet、system pod (kube-proxy, CNI等) は無視する

## GangJob ライフサイクル

```
Pending → Running → Succeeded (全Pod成功)
                  → Failed    (1Pod失敗 → 残りPod削除 → リトライなし)
        → Failed (バリデーションエラー: numNodes > capacity等。PodGroup/Podは作成しない)
```

## フロー

```
User: GangJob作成
  → Controller: バリデーション (numNodes vs Queue.capacity/maxNodesPerJob)
    → NG → GangJob.status.phase=Failed, reason記録, Event発行 (終了。PodGroup/Podは作成しない)
    → OK → PodGroup + Pod群作成 (schedulerName: gang-scheduler, nodeName空)
  → Scheduler: Queue内でPodGroup評価 → 条件OK → Nodeにtaint付与 → Binding API (`/pods/<name>/binding`) でbind
  → kubelet: nodeName検知 → コンテナ起動
  → 全Pod成功 → Controller: taint除去 → GangJob Succeeded
  → 1Pod失敗 → Controller: 失敗Pod情報をGangJob.statusに記録(永続化確認) → 残Pod削除 → taint除去 → GangJob Failed
```

## Queueのノード割当

- 静的。マニフェストで管理し、apply時に反映

## テスト環境

- kind (master 3台 + worker 24台)
- jobは `sleep n` + flag で Succeeded/Failed をシミュレート

## スコープ外

- kube-scheduler Plugin方式
- Preemption
- GPU単位リソース管理
- Fair sharing / Queue間リソース貸し借り
- 失敗時リトライ
- ネットワークトポロジ考慮（同一ラック配置等）
- Metrics（Prometheus連携等。v1では不要、運用後に追加検討）

## 異常系/エッジケースにおける振る舞い

### Bind失敗時のロールバック

N個のPodをbindする途中で1つ失敗した場合、bind済みのPodを削除し、付与済みtaintを除去し、Controllerが同数の新Podを再作成してPodGroup全体をPendingに戻す。次のスケジューリングサイクルで再評価する。一度bindしたPodはKubernetes API上unbindできないため、削除→再作成が唯一の手段。

### Scheduler/Controllerクラッシュリカバリ

起動時にGangJob/PodGroup/Podの現在の状態を読み取り、reconciliationを行う。部分bind状態（N個中k個bind済み、PodGroupがまだRunningでない）を検出した場合は、bind済みPodを全て削除し、Controllerが新Podを再作成してPendingに戻す。all-or-nothingを徹底し、中間状態を残さない。

孤立taintの回収: `gang.k8s.io/exclusive` taintが付与されているが対応するgang Podが存在しないNodeを検出し、taintを除去する。クラッシュによるtaint残存でNodeが永久に使えなくなることを防ぐ。起動時のreconciliationおよび定期re-syncの両方で実行する。

### Node障害時の扱い

通常のPod失敗と同じフロー（失敗Pod情報をGangJob.statusに記録 → 残りPod削除 → taint除去 → GangJob Failed）。Schedulerは空きNode計算時にNodeのReady conditionを確認し、NotReady/UnknownなNodeは候補から除外する。NIC故障等のk8sが検知できない障害は別の仕組み（node error handler）の責務でスコープ外。

### GangJob削除/キャンセル

ownerReferenceによるcascade deleteに任せる。GangJob削除 → PodGroup + Pod群が削除。特別な処理は不要。

### Schedulerのループ方式

イベント駆動（PodGroup作成/Pod状態変化/Node状態変化のwatch）+ 定期re-syncの併用。k8s informerの標準パターンに従う。

### Nodeの二重割当防止

Queue単位で独立してスケジューリングルール（同priority内FIFO + head-of-line blocking + 高priorityのみスキップ可）に従い評価する。1サイクル内で割り当て済みNodeをインメモリで追跡し、Queue間のNode重複に対応する。単一goroutineで処理するためロックは不要。

### maxRuntimeの強制

ControllerがRunning状態のGangJobを監視し、maxRuntime超過で全Pod削除 → taint除去 → GangJob Failedにする。maxRuntimeの起点はPodGroup phaseがRunningに遷移した時刻（＝全PodのbindをSchedulerが完了し、PodGroupをRunningに更新した時点）。bind待ち時間は含まない。

### 完了したGangJobの寿命

完了（Succeeded/Failed）したGangJobは3ヶ月後に自動削除（TTL）。Podはジョブ完了/失敗時に即削除される（ログは外部ログ基盤で参照。失敗情報はGangJob.statusに記録済み）。

> NOTE: K8s標準の `TTLAfterFinished` はJobリソース専用でCRDには適用されない。Controllerに定期的な掃除reconciliation（完了から3ヶ月経過したGangJob/PodGroupを削除）を実装する。

### FIFO順序

同priority内のFIFO順序は `metadata.creationTimestamp` で決定。Namespace横断で適用される。同一timestampの場合は `metadata.uid` の辞書順でtie-breakする。

### Pod失敗の定義

Pod phaseが`Failed`になったものを失敗とみなす。コンテナのexit code等の詳細は見ず、Pod phaseのみで判断する。

### Node割当アルゴリズム

空きNodeの中からどのNodeにPodを配置するかは、Node名の辞書順で決定する。ネットワークトポロジ（同一ラック配置等）の考慮はスコープ外。

## 観測性

### Events

ControllerとSchedulerはK8s Eventsを発行し、ユーザーが `kubectl describe` で状態を追跡できるようにする。Eventはphaseまたはconditionの遷移時のみ発行する（reconciliation loopでの重複発行を防ぐ）。

| 対象 | Event | タイミング |
|------|-------|----------|
| GangJob | `ValidationFailed` | バリデーションエラーでFailed遷移 |
| GangJob | `PodGroupCreated` | PodGroup + Pod群作成完了 |
| GangJob | `JobRunning` | 全Podがbindされ Running遷移 |
| GangJob | `JobSucceeded` | 全Pod成功 |
| GangJob | `JobFailed` | Pod失敗検知 |
| GangJob | `MaxRuntimeExceeded` | maxRuntime超過で強制終了 |
| PodGroup | `ScheduleSucceeded` | 全Podのbind完了 |
| PodGroup | `ScheduleFailed` | bind失敗（空きNode不足等） |
| PodGroup | `BindRollback` | 部分bind失敗によるロールバック |

### PodGroup Conditions

```yaml
status:
  phase: Pending
  conditions:
    - type: Schedulable
      status: "False"
      reason: InsufficientNodes
      message: "required 4 nodes, 2 available in queue research-team-a"
      lastTransitionTime: "2026-05-24T10:00:00Z"
```

- `Schedulable` — スケジュール可能かどうか。空きNode不足時にFalse + 理由を表示。各スケジューリングサイクルで評価し、空きNodeが確保できたらTrueに遷移する

## CRDスキーマ

### Queue

```yaml
apiVersion: gang.k8s.io/v1alpha1
kind: Queue
metadata:
  name: research-team-a
spec:
  nodeSelector:
    matchLabels:
      gang.k8s.io/queue.research-team-a: "true"
  capacity: 16
  maxNodesPerJob: 8
  serverType: gpu
```

- `nodeSelector` — Queue配下のNodeをlabelで指定。1 Nodeが複数Queueに所属可能
- `capacity` — 組織に割り当てられたノード数（ユーザー確認用）
- `maxNodesPerJob` — 1ジョブあたりの最大ノード数。GangJob作成時に `numNodes > maxNodesPerJob` なら弾く
- `serverType` — `gpu` or `cpu`。情報用フィールド、スケジューリングには使わない

### Node（label設計）

```yaml
apiVersion: v1
kind: Node
metadata:
  name: worker-01
  labels:
    gang.k8s.io/queue.research-team-a: "true"
    gang.k8s.io/queue.shared-pool: "true"
```

- labelキーに `gang.k8s.io/queue.<queue名>`、valueは `"true"`
- 複数Queueに所属する場合はlabelを複数付与

### GangJob

```yaml
apiVersion: gang.k8s.io/v1alpha1
kind: GangJob
metadata:
  name: training-job-001
  namespace: user-alice
spec:
  queueName: research-team-a
  numNodes: 4
  priority: 1
  maxRuntime: 24h
  ignoreCordoned: false
  template:
    spec:
      containers:
        - name: worker
          image: training:latest
```

- `queueName` — 投入先のQueue
- `numNodes` — 確保するノード数（きっちりこの数）
- `priority` — 1-10、デフォルト1、高い方が優先。隠しオプション
- `maxRuntime` — デフォルト24h、隠しオプションで延長可
- `template` — 全Podに適用される単一テンプレート
- `ignoreCordoned` — デフォルトfalse。trueでcordonされたNodeもスケジュール対象にする。隠しオプション（メンテナンス前にNodeの動作確認ジョブを流す用途）
- バリデーション: `numNodes > Queue.maxNodesPerJob` または `numNodes > Queue.capacity` ならController reconciliation時にGangJob.status.phaseをFailedにしてEventで通知する（Validating Webhookは証明書管理等の運用負荷が大きいため不採用）
- status:
  - `phase` — `Pending | Running | Succeeded | Failed`
  - `reason` — Failed時の原因区分: `ValidationError | PodFailed | MaxRuntimeExceeded`
  - `message` — 人間可読な失敗詳細（例: `"numNodes 8 exceeds queue maxNodesPerJob 4"`, `"pod training-job-001-3 failed: OOMKilled"`）
  - `failedPod` — 失敗起因のPod名（reason=PodFailedの場合のみ）
  - `runningStartTime` — PodGroup Running遷移時刻（maxRuntime計算の起点）

### PodGroup

```yaml
apiVersion: gang.k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: training-job-001
  namespace: user-alice
  ownerReferences:
    - apiVersion: gang.k8s.io/v1alpha1
      kind: GangJob
      name: training-job-001
spec:
  queueName: research-team-a
  numNodes: 4
  priority: 1
  ignoreCordoned: false
status:
  phase: Pending  # Pending | Running
```

- Controllerが GangJob検知時に作成。specはGangJobからのコピー
- SchedulerはPodGroupのみで判断可能（GangJob参照不要）
- ジョブ完了/失敗時にPodGroupもGangJobと同じ3ヶ月TTLで残す（デバッグ時にPodGroupのstatus/conditionsを参照するため）
- ownerReference: GangJob → PodGroup、GangJob → Pod群（並列。PodGroupはPodを所有しない）
- PodGroupとPodはlabel `gang.k8s.io/podgroup: <podgroup名>` で紐付け

### リソーススコープ

| リソース | スコープ |
|---------|---------|
| Queue | Cluster-scoped |
| Node | Cluster-scoped |
| GangJob | Namespaced |
| PodGroup | Namespaced |
| Pod | Namespaced |

## 未決事項

- [x] Queue → Node 紐付けの具体的なlabel設計 → `gang.k8s.io/queue.<queue名>: "true"`
- [x] 異常系/エッジケースにおける振る舞い
  - [x] Bind失敗時のロールバック
  - [x] Scheduler/Controllerクラッシュリカバリ
  - [x] Node障害時の扱い
  - [x] GangJob削除/キャンセル
  - [x] Schedulerのループ方式
  - [x] Nodeの二重割当防止
  - [x] Pod失敗の定義
- [x] Bind方式 → Binding API (`/pods/<name>/binding`)
- [x] ロールバック方式 → Pod削除→再作成（unbind不可のため）
- [x] 「使用中Node」の定義 → gang-schedulerがbindしたPodが存在するNode
- [x] Pod終了方式 → Pod削除（ログは外部基盤、失敗情報はGangJob.statusに記録）
- [x] FIFO tie-breaker → `metadata.uid` 辞書順
- [x] maxRuntime起点 → PodGroup Running遷移時刻
- [x] PodGroup-Pod紐付けlabel → `gang.k8s.io/podgroup: <podgroup名>`
- [x] 観測性 → Events + PodGroup Conditions
- [x] バリデーション方式 → Controller reconciliation（Webhook不採用）
- [x] taint操作順序 → taint先行 → bind。孤立taint回収をreconciliationに含む
- [x] 外部ログ基盤前提 → 前提条件に明記
- [x] validation失敗パス → ライフサイクル図/フロー図に追加、GangJob.status.reason/messageで汎用化
- [ ] K8s PodGroup schedulingがGAになった場合の移行方針

## 参考ドキュメント

- [Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
- [K8s v1.36 Workload-Aware Scheduling Blog](https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/)

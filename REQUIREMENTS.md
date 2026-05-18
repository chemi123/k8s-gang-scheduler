# K8s Gang Scheduler 要件定義

## 背景

- 分散学習基盤でVolcanoを使用中。コードが煩雑で用途に対してリッチすぎるため自作する
- K8s v1.36でPodGroup schedulingがalpha導入されているが、インターフェース変更が予想されるため自作しつつ概念レベルで寄せる

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
3. **Node単位割当** — GPUではなくNode単位でリソース確保
4. **同priority内FIFO + head-of-line blocking** — 先頭が詰まったら同priority以下の後続は全て待つ
5. **高priorityのみキュー追い越し可** — 後続jobのpriorityが先頭より厳密に高い場合のみ抜かせる

## Node-Queue関係

- 1 Nodeは複数Queueに所属可能
- Queue CRDがnodeSelectorを持つ（Queue側でNodeを指定する方式）
- Available nodeの計算はQueue問わず「現在使用中でないNode」で判断

## GangJob ライフサイクル

```
Pending → Running → Succeeded (全Pod成功)
                  → Failed    (1Pod失敗 → 残りPod終了、Pod自体は削除せず残す。リトライなし)
```

## フロー

```
User: GangJob作成
  → Controller: PodGroup + Pod群作成 (schedulerName: gang-scheduler, nodeName空)
  → Scheduler: Queue内でPodGroup評価 → 条件OK → Pod.spec.nodeNameセット (bind)
  → kubelet: nodeName検知 → コンテナ起動
  → 全Pod成功 → Controller: GangJob Succeeded
  → 1Pod失敗 → Controller: 残Pod終了(削除せず残す) → PodGroup.status.failedPodに記録 → GangJob Failed
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

## 異常系/エッジケースにおける振る舞い

### Bind失敗時のロールバック

TODO

### Scheduler/Controllerクラッシュリカバリ

TODO

### Node障害時の扱い

TODO

### GangJob削除/キャンセル

TODO

### Schedulerのループ方式

TODO

### Nodeの二重割当防止

TODO

### Pod失敗の定義

TODO

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
- `maxNodesPerJob` — 1ジョブあたりの最大ノード数。GangJob作成時に `minNodes > maxNodesPerJob` なら弾く
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
- バリデーション: `numNodes > Queue.maxNodesPerJob` または `numNodes > Queue.capacity` なら作成時に弾く

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
status:
  phase: Pending  # Pending | Running | Succeeded | Failed
  failedPod: ""   # 失敗起因のPod名（Failedの場合のみ）
```

- Controllerが GangJob検知時に作成。specはGangJobからのコピー
- SchedulerはPodGroupのみで判断可能（GangJob参照不要）
- ownerReference: GangJob → PodGroup → Pod群 のチェーンでcascade delete

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
- [ ] 異常系/エッジケースにおける振る舞い
  - [ ] Bind失敗時のロールバック
  - [ ] Scheduler/Controllerクラッシュリカバリ
  - [ ] Node障害時の扱い
  - [ ] GangJob削除/キャンセル
  - [ ] Schedulerのループ方式
  - [ ] Nodeの二重割当防止
  - [ ] Pod失敗の定義
- [ ] K8s PodGroup schedulingがGAになった場合の移行方針

## 参考ドキュメント

- [Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
- [K8s v1.36 Workload-Aware Scheduling Blog](https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/)

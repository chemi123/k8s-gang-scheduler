# REQUIREMENTS.md レビュー対応まとめ

Opus 4.7 による REQUIREMENTS.md レビュー（ISSUES.md: 18項目）に対する対応結果。

## 対応方針

- 設計上の曖昧さや未定義事項を具体化
- Kubernetes API の制約に沿った方式に修正
- 観測性など欠落していたセクションを追加

## 対応詳細

### 🔴 設計上のクリティカル（4件）

| # | 指摘 | 対応 |
|---|------|------|
| 1 | Bind方式が未定義。nodeName直接patchはrace/権限面で問題。ロールバック（unbind）も仕様上不可能 | Binding API (`/pods/<name>/binding`) に明記。ロールバックはPod削除→再作成に変更（unbind不可のため） |
| 2 | 「使用中でないNode」の定義が曖昧 | 「gang-schedulerがbindしたPodが存在するNode」と定義。DaemonSet/system podは無視 |
| 3 | Node単位割当だがkubeletがリソース不足で弾く可能性 | bindと同時にtaint `gang.k8s.io/exclusive=true:NoSchedule` を付与し、kube-schedulerからの配置を防止 |
| 4 | 「残Pod終了(削除せず残す)」の具体手段が未定義 | Pod削除に変更。失敗情報はPod削除前にGangJob.status (`failedPod`, `failedReason`) に記録。ログは外部ログ基盤で参照 |

### 🟡 ルール/整合性（5件）

| # | 指摘 | 対応 |
|---|------|------|
| 5 | 高priority連続投入で低priorityがstarvation | 意図通り。「starvation許容（aging等の救済策は設けない）」を明記 |
| 6 | FIFO同着時のtie-breaker未定義 | `metadata.uid` の辞書順でtie-break |
| 7 | maxRuntime起点が不明確 | PodGroup phaseがRunningに遷移した時刻。bind待ち時間は含まない。GangJob.statusに `runningStartTime` 追加 |
| 8 | PodGroup即削除 vs Pod 3ヶ月TTLの不一致 | PodGroupもGangJobと同じ3ヶ月TTLに変更。Podはジョブ完了/失敗時に即削除 |
| 9 | cascade deleteで記録が残らない | 対応なし（現状維持）。ownerReferenceによるcascade deleteはK8s標準動作であり、Running中の意図的削除はユーザー操作として許容 |

### 🟢 仕様の隙間（7件）

| # | 指摘 | 対応 |
|---|------|------|
| 10 | PodGroupとPodの紐付けlabel未定義 | `gang.k8s.io/podgroup: <podgroup名>` に確定 |
| 11 | schedulerName付与の責務が不明 | 対応不要。REQUIREMENTS.md L56で「Controller: Pod群作成 (schedulerName: gang-scheduler)」と既に定義済み |
| 12 | priority範囲1-10が狭い | 対応なし（現状維持）。CRD独自フィールドであり用途上10段階で十分 |
| 13 | PodGroup specがGangJobの単純コピーで存在意義が薄い | 対応なし（現状維持）。K8s v1.36のWorkload/PodGroup分離に概念を寄せた設計意図が既に記載済み |
| 14 | ignoreCordonedの動作確認価値が不明 | 用途を「メンテナンス前にNodeの動作確認ジョブを流す用途」と明確化 |
| 15 | 観測性が未定義 | 「観測性」セクション新設。Events（8種）+ PodGroup Conditions (`Schedulable`) を定義 |
| 16 | リカバリの「あるべき状態」が曖昧（部分bind時） | 部分bind検出→bind済みPod全削除→再作成→Pendingに戻す、と明記。all-or-nothingを徹底 |

### 🔵 運用/テスト（2件）

| # | 指摘 | 対応 |
|---|------|------|
| 17 | kind worker 24台は重い | 対応なし。実機確認は実装フェーズで判断 |
| 18 | Validating Webhookの要否 | → 第2ラウンド#23で方針変更。Controller reconciliationに変更 |

### レビュー外の追加対応

| 項目 | 内容 |
|------|------|
| Node割当アルゴリズム | Node名の辞書順で決定。トポロジ考慮はスコープ外 |
| GangJob.status拡充 | `failedReason`（失敗詳細）、`runningStartTime`（maxRuntime起点）を追加 |
| スコープ外の明示 | ネットワークトポロジ考慮、Metrics（Prometheus連携）を明示的に除外 |

## 第1ラウンド集計

- 全18件中、**12件を反映**（修正 or 明確化）
- 4件は対応不要（既に定義済み or 現状維持が妥当）
- 2件は実装フェーズで判断

---

## 第2ラウンド: レビュー対応後の追加指摘（7件）

第1ラウンドの対応によって新たに浮上した懸念への対応。

### 🔴 設計上のクリティカル（2件）

| # | 指摘 | 対応 |
|---|------|------|
| 19 | taint操作の順序が未定義。bind先行だとrace window、クラッシュでtaint残存→Node永久使用不能 | taint先行→bindの順序を明記。クラッシュリカバリに孤立taint回収ロジック（`gang.k8s.io/exclusive` taintがあるが対応Podが無いNodeを検出→taint除去）を追加 |
| 20 | failedReason記録→Pod削除の順序が暗黙的。間でクラッシュすると情報喪失 | フローに「永続化確認」を追記。標準的なreconciliationの操作順序として自然なので、詳細は実装に委ねる |

### 🟡 ルール/整合性（2件）

| # | 指摘 | 対応 |
|---|------|------|
| 21 | Pod即削除は外部ログ基盤を暗黙の前提にしている | 「前提条件」セクションを新設し、外部ログ基盤（Loki, CloudWatch等）の稼働を明記 |
| 22 | TTLAfterFinishedはJobリソース専用でCRDに適用されない | Controller内の自前TTL reconciliationが必要な旨をNOTEとして追記 |

### 🟢 仕様の隙間（3件）

| # | 指摘 | 対応 |
|---|------|------|
| 23 | Validating WebhookはTLS証明書管理等の運用負荷が大きい | Webhook不採用に方針変更。Controller reconciliation時にGangJob.statusをFailedにしてEventで通知する方式に |
| 24 | reconciliation loopで毎回Event発行すると重複爆発 | 「phase遷移時のみ発行」のルールを追記 |
| 25 | PodGroup Conditionsの遷移ルール未定義 | 各スケジューリングサイクルで評価し状態に応じてTrue/False遷移する旨を追記 |

## 第2ラウンド集計

- 全7件中、**7件すべて反映**
  - 設計変更: 3件（#19 taint順序、#21 前提条件明記、#23 Webhook→Controller validation）
  - NOTE/軽微追記: 4件（#20, #22, #24, #25）

---

## 第3ラウンド: Webhook → Controller validation 変更の波及（5件）

第2ラウンド#23の方針変更によって生じた整合性の抜けへの対応。

### 🟡 ルール/整合性（3件）

| # | 指摘 | 対応 |
|---|------|------|
| 26 | ライフサイクル図にvalidation failedパスがない | `Pending → Failed (バリデーションエラー)` パスを追加 |
| 27 | フロー図にvalidation stepが抜けている | Controller処理の先頭にバリデーション分岐を挿入。NG時はPodGroup/Pod作成せずFailed |
| 28 | failedPod/failedReasonがPod失敗前提で、validation失敗時に宙に浮く | statusフィールドを再設計。`reason`（原因区分: `ValidationError / PodFailed / MaxRuntimeExceeded`）+ `message`（人間可読な詳細）に汎用化。`failedPod`はreason=PodFailed時のみ使用 |

### 🟢 細かい点（2件）

| # | 指摘 | 対応 |
|---|------|------|
| 29 | 「phase遷移時のみ」だとScheduleFailed Event（phaseはPendingのまま）が出ない | 「phase**または**conditionの遷移時」に修正 |
| 30 | 孤立taint回収がクラッシュリカバリ時のみか不明確 | 「起動時のreconciliationおよび定期re-syncの両方で実行」と明記 |

## 第3ラウンド集計

- 全5件中、**5件すべて反映**
  - 設計変更: 3件（#26 ライフサイクル図、#27 フロー図、#28 statusフィールド再設計）
  - 軽微修正: 2件（#29, #30）

## 全体集計（第1+第2+第3ラウンド）

- 全30件中、**24件を反映**
- 4件は対応不要（既に定義済み or 現状維持が妥当）
- 2件は実装フェーズで判断

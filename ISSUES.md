# REQUIREMENTS.md レビュー指摘事項

REQUIREMENTS.md の現状に対する設計上の懸念点リスト。実装着手前に方針決定が必要。

## 🔴 設計上のクリティカル

### 1. Bindの実装方法が未定義
- フローに「Pod.spec.nodeNameセット (bind)」とあるが、本来は **Binding API (`/pods/<name>/binding`)** を使うのが正攻法。直接patchはraceや権限面で問題が出やすい。
- これに関連して「bind失敗時にnodeNameを外す」ロールバックも、Binding API的には **一度bindしたPodは未bindに戻せない**（仕様上Pod削除→再作成が必要）。ロールバック方式の再検討が必要。

### 2. 「使用中でないNode」の定義が曖昧 (REQUIREMENTS.md L43)
他schedulerと共存する設計なのに、以下が未定義:
- kube-schedulerが配置した一般Podが乗っているNodeは「使用中」扱い？
- DaemonSet / system pod (kube-proxy, CNI等) は無視？
- 答え次第で空きNode計算ロジックが大きく変わる。

### 3. Node単位割当 vs Pod resources の整合性
- `template.spec.containers` には通常resources指定がある。Node単位確保なのに、kubeletがNode resource不足で弾く可能性は？
- 「Nodeを独占する」前提なら、bindと同時にtaintを打つ等が必要では。

### 4. 「残Pod終了(削除せず残す)」の具体手段 (REQUIREMENTS.md L50, L60)
- Pod削除なしで「終了」させる方法が未定義。`activeDeadlineSeconds`設定 / containerにSIGTERM送信 / status直接書き換え、のどれ？
- Pod削除しないと結局Pod listが膨らみ続ける（TTL 3ヶ月との関係も）。

## 🟡 ルール/整合性

### 5. Head-of-line blocking + 高priorityスキップの starvation
- 「同priority以下は待つ」「高priorityのみ抜かせる」の組み合わせ → 高priorityが流れ続ければ低priorityは **永遠に実行されない**。意図通り？aging等の救済策は不要？

### 6. FIFO同着時のtie-breaker未定義 (REQUIREMENTS.md L116)
- `creationTimestamp`は秒粒度。同秒投入時の順序が不定。`uid` か `name` でtie-break明示すべき。

### 7. maxRuntime起点の定義
- 「Running状態超過」とあるが、起点は **PodGroup Running遷移時 / 最初のPod Running時 / 全Pod Running時** のどれ？bind待ち時間を含む/含まないで大きく違う。

### 8. PodGroup削除タイミング vs Pod 3ヶ月TTL の不一致 (REQUIREMENTS.md L112, L215)
- 完了時にPodGroupは即削除、Podは3ヶ月残す。デバッグ時にPodGroupが既に無い状態でPodだけ残るのは扱いにくい。PodGroupも3ヶ月残す方が一貫性あり。

### 9. GangJob削除 cascade vs failedPod記録 (REQUIREMENTS.md L96)
- cascade deleteで全消去 → Running中に削除されたら **何も記録に残らない**。「キャンセル」と「Failure」の区別は不要？

## 🟢 仕様の隙間

### 10. PodGroupとPodの紐付けlabel未定義 (REQUIREMENTS.md L217)
- 「labelで紐付け」とだけあるが、key/value仕様を明示すべき（例: `gang.k8s.io/podgroup: <name>`）。

### 11. schedulerName付与の責務
- Controllerが自動付与？Pod template側に書かせる？未定義。

### 12. priority範囲が1-10と狭い (REQUIREMENTS.md L184)
- K8s標準は `int32`。1-10と限定する理由が不明。後の拡張で困る可能性。

### 13. PodGroup specの存在意義
- GangJobからの単純コピー。なら **Scheduler が直接GangJob見れば良いのでは**？という疑問が残る。「Scheduler責務をPodGroupだけに閉じる」のメリットが、二重管理コストに見合うか要再考。

### 14. cordoned Node判定とignoreCordonedの動作確認価値 (REQUIREMENTS.md L186)
- cordonはschedulerへのhint。Schedulerが無視すればbindできるが、それは「cordon無効化」と同義。「動作確認用」として何を確認したいか不明。

### 15. 観測性が未定義
- Events / Metrics / Conditions の設計言及なし。「なぜbindできないか」をユーザーがどう知るか不明。最低限PodGroup.statusにconditions欲しい。

### 16. リカバリの「あるべき状態」が曖昧 (REQUIREMENTS.md L88)
- 部分bind状態（N個中k個bind済み）でクラッシュ→起動時にどう判断？「残りbind継続」「全部unbind」のどちら？

## 🔵 運用/テスト

### 17. kind worker 24台は重い (REQUIREMENTS.md L69)
- Docker上に27コンテナ。手元マシンで実用に耐えるか確認推奨。

### 18. Validating Webhook の要否
- `numNodes > capacity` バリデーション (REQUIREMENTS.md L187) はAPI level？Controller reconciliation？前者ならadmission webhook実装が増える。

---

優先度: **1, 2, 4** は実装で詰まる可能性が高いため最優先で方針決定。

---

# 第2ラウンド: レビュー対応後の追加指摘

REVIEW_RESPONSE.md / REQUIREMENTS.md 更新版に対する追加レビュー。対応によって新たに浮上した懸念。

## 🔴 設計上のクリティカル

### 19. taint操作の順序とクラッシュ耐性

bindとtaint付与の順序が未定義:
- **bind先行** → bind直後〜taint付与までの隙にkube-schedulerが別Podを乗せる可能性
- **taint先行** → bind失敗時にtaint除去も必要、ロールバックが2段階に
- Controllerクラッシュ → **taint残存でNodeが永久に使えない**リスク
- リカバリで「PodGroup未関連のtaintを除去」するreconciliationロジックが必要。要件への明記推奨。

### 20. failedReason記録のatomicity (REQUIREMENTS.md L130, L237)

「Pod削除前にControllerが記録」とあるが順序が暗黙的:
1. `GangJob.status.failedReason` write
2. patch成功確認
3. Pod削除

の順序を明示しないと、間でクラッシュした場合 **失敗Podが消えてreason取得不能** になる。Pod削除前にstatus更新が永続化されることを保証する記述が必要。

## 🟡 ルール/整合性

### 21. Pod即削除はデバッグ性を犠牲にしすぎ (REQUIREMENTS.md L117)

> 「Podはジョブ完了/失敗時に即削除される。ログは外部ログ基盤で参照」

- **前提として外部ログ基盤（Loki/CloudWatch等）の存在が必須** だが、要件に明示されていない
- Succeeded も即削除する必要は薄い。数時間〜数日残せば `kubectl logs` で追える
- 「外部ログ基盤前提」を **スコープ条件として明記**するか、短期Pod保持期間を設けるか検討余地あり

### 22. PodGroup TTL 3ヶ月の実装手段

- K8s標準の `TTLAfterFinished` は **Jobリソース専用**。CRD（PodGroup/GangJob）には適用されない
- Controller内に自前TTL reconciliationロジックが必要 → 実装項目として明示すべき

## 🟢 仕様の隙間

### 23. Validating Webhook の運用前提

- Webhook運用には **TLS証明書管理**（cert-manager等）が必要
- kind環境でWebhookを動かす手順も結構重い
- 代替として **Controller reconciliationで弾く** 選択肢もある（admission前のリジェクト不可だがUX差は小さい）。トレードオフの再確認推奨。

### 24. Event発行の重複防止ルール

- reconciliation loopで毎回Event発行すると重複爆発
- 「state transition時のみ発行」等のルール明記推奨

### 25. PodGroup Conditions の寿命管理

- `Schedulable: False` (理由: InsufficientNodes) になった後、空きが増えたらTrueに戻す必要
- conditionの遷移ルール（追加/更新/削除のタイミング）が未定義

---

## 第2ラウンド集計

| 重要度 | 項目 |
|---|---|
| 🔴 | 19. taint操作順序とクラッシュ時のtaint残存ハンドリング |
| 🔴 | 20. failedReason記録 → Pod削除のatomicity |
| 🟡 | 21. Pod即削除の前提（外部ログ基盤）を要件に明記 |
| 🟡 | 22. PodGroup/GangJob TTL は自前実装が必要なことを明示 |
| 🟢 | 23. Validating Webhook運用負荷の認識 |
| 🟢 | 24. Event発行の重複防止ルール |
| 🟢 | 25. PodGroup Conditionsの遷移ルール |

---

# 第3ラウンド: Webhook → Controller validation 変更の波及

第2ラウンド#23の方針変更（Validating Webhook不採用 → Controller reconciliationでvalidation）によって生じた整合性の抜け。

## 🟡 ルール/整合性

### 26. ライフサイクル図に validation failed パスがない (REQUIREMENTS.md L42-44)

現状:
```
Pending → Running → Succeeded
                  → Failed (1Pod失敗)
```

Controller validationでFailedになるケース（`numNodes > capacity/maxNodesPerJob`）が反映されていない。本来:
```
Pending → Running → Succeeded
                  → Failed (Pod失敗, maxRuntime超過)
        → Failed (バリデーションエラー: 一度もRunningにならない)
```

### 27. フロー図に validation step が抜けている (REQUIREMENTS.md L57-65)

現状のフローは validation step を含まない:
```
User: GangJob作成
  → Controller: PodGroup + Pod群作成 ...
```

Controller reconciliationでvalidationする方針なら、validation step を挿入し **NG時はPod作成しない** ことを明示すべき:
```
User: GangJob作成
  → Controller: validation
    → NG → GangJob.status.phase=Failed → Event発行 (停止)
    → OK → PodGroup + Pod群作成 ...
```

### 28. Controller validation 失敗時の `failedReason` 取り扱い

GangJob.statusスキーマ (REQUIREMENTS.md L235-238):
```
- failedPod   — 失敗起因のPod名（Failedの場合のみ）
- failedReason — 失敗Pod のstatus.reason / message / exit code
```

両方とも **Pod失敗起因の前提** で定義されている。Validation失敗時は失敗Podが存在しないため、これらフィールドの記載が宙に浮く。
- `failedReason` を「Pod失敗 or validation失敗の理由」に拡張
- もしくは別フィールド `validationError` を追加
- いずれにせよ「Validation失敗時にどうstatusを書くか」を明示すべき

## 🟢 細かい点

### 29. Event発行ルール「phase遷移時のみ」の解釈

`ScheduleFailed` は空きNode不足等で発生するが、 **phaseは Pending のまま** （Conditionだけ更新される）。「phase遷移時のみ発行」を厳密適用すると `ScheduleFailed` Event が一度も出ない。

「state transition」を **phase or condition の遷移時** と明確化するか、Event別にルールを定義するのが安全。

### 30. 孤立taint回収の発火タイミング

「クラッシュリカバリ時」とあるが、 **定期re-sync時** にも実行されるか不明確。実装上は定期re-syncで回せば自然に回収されるはずだが、明示があると安心。

---

## 第3ラウンド集計

| 重要度 | 項目 |
|---|---|
| 🟡 | 26. ライフサイクル図に validation failed パス追加 |
| 🟡 | 27. フロー図に validation step 挿入 |
| 🟡 | 28. validation失敗時の status 記述方法 |
| 🟢 | 29. Event発行ルールの「state transition」定義 |
| 🟢 | 30. 孤立taint回収を定期re-syncでも実行する旨明示 |

26〜28 は Webhook→Controller validation 変更の波及。3つセットで反映すると整合する。

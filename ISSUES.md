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

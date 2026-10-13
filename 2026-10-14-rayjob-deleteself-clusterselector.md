# KubeRay PR 筆記：DeleteSelf 與 clusterSelector

日期：2026-10-14  
PR commit：`5f68a08f`（`[RayJob] Fix DeleteSelf with clusterSelector`）

## 1. 為什麼移動 clusterSelector 判斷就能修好？

原本 RayJob 到達終止狀態後，只要設定了 `clusterSelector`，controller 就提前 return，連 deletion policy 都不執行。因此 `DeleteSelf` 無法刪除 RayJob。

修改後先處理 deletion strategy，再檢查 `clusterSelector`：

```text
有啟用 RayJobDeletionPolicy 且設定 deletionStrategy
  → 執行 deletion policy，直接 return
否則
  → 有 clusterSelector 就跳過 shutdownAfterJobFinishes 清理
```

不是「先刪 RayJob，再靠後面的 clusterSelector 判斷保護 RayCluster」；policy 分支已經 return，不會走到後面的判斷。

RayCluster 得以保留的原因：

- `DeleteSelf` 只呼叫 `Client.Delete(ctx, rayJob)`。
- clusterSelector 指定的共享 cluster 不會被加上指向這個 RayJob 的 owner reference，所以不會因這個 RayJob 被刪除而連帶回收。
- `deleteClusterResources()` 本身也有 clusterSelector 保護。
- validation 不允許 clusterSelector 搭配 DeleteCluster 或 DeleteWorkers。

## 2. 既有 cluster 與 owner reference

正常使用 RayJob API 時：

| 設定 | 行為 |
| --- | --- |
| `rayClusterSpec` | controller 建立專屬 RayCluster，設定指向 RayJob 的 owner reference |
| `clusterSelector` | 指定使用既有 RayCluster，不設定指向此 RayJob 的 owner reference |

controller 建立 cluster 時會呼叫：

```go
ctrl.SetControllerReference(rayJobInstance, rayCluster, r.Scheme)
```

`clusterSelector` 裡的 `ray.io/cluster: processing` 在這段流程中是指定 cluster 名稱 `processing`，不是任意 label selector 查詢。

## 3. fake client 的設定

```go
fakeClient := clientFake.NewClientBuilder().
    WithScheme(newScheme).
    WithObjects(rayJob, rayCluster).
    WithStatusSubresource(rayJob).
    Build()
```

### WithScheme(newScheme)

```go
newScheme := runtime.NewScheme()
require.NoError(t, rayv1.AddToScheme(newScheme))
```

Scheme 註冊 Go 型別與 Kubernetes Group/Version/Kind 的對應，讓 fake client 認得 RayJob、RayCluster。

不一定要建立新的 Scheme，也可以使用已註冊的 Scheme；這裡建立新的能隔離測試。不能在沒有其他註冊來源的情況下，直接認為拿掉也沒差。

### WithObjects(rayJob, rayCluster)

將準備好的物件放進 fake client，包括測試預先設定的 `Succeeded`、`EndTime` 等 status 資料。

### WithStatusSubresource(rayJob)

宣告 RayJob 這個型別有獨立的 status subresource，模擬真實 API 的更新語意：

```go
client.Update(ctx, rayJob)          // 不更新 status
client.Status().Update(ctx, rayJob) // 更新 status，保留其他欄位
```

它不是初始化 status，也不是讓 DeleteSelf 生效的開關。

就這支測試目前走的路徑而言，沒有更新 status，因此依程式流程判讀，加不加不影響預期結果；保留能更符合真實 API 行為。此結論未透過移除設定後實跑驗證。

## 4. 為什麼呼叫兩次 Reconcile？finalizer 一定要加嗎？

第一次 Reconcile：

1. 執行 DeleteSelf，呼叫 Delete。
2. 因為 RayJob 還有 finalizer，物件仍存在，但已設定 DeletionTimestamp。
3. 測試確認 DeletionTimestamp 有值。

第二次 Reconcile：

1. 發現 DeletionTimestamp，進入刪除處理。
2. Job 已 Succeeded，不需要再呼叫 StopJob。
3. 移除 finalizer 並 Update。
4. fake client 移除物件，測試確認 Get 回傳 NotFound。

正常 controller 會在 RayJob 初始化時加 finalizer；這個測試直接建立已完成的 RayJob，所以手動補上，模擬真實生命週期。

若只想測 DeleteSelf 是否刪除 RayJob，可以不加 finalizer，但第一次 Reconcile 後就要檢查 NotFound。現有測試的兩階段斷言需要 finalizer。

## 5. NamespacedName 是什麼？

```go
request := reconcile.Request{
    NamespacedName: client.ObjectKeyFromObject(rayJob),
}
```

代表 namespace 與 name 的組合：

```go
types.NamespacedName{
    Namespace: "default",
    Name:      "test-rayjob",
}
```

```go
assert.True(t, apierrors.IsNotFound(
    fakeClient.Get(ctx, request.NamespacedName, &rayv1.RayJob{}),
))
```

`&rayv1.RayJob{}` 指定查詢的資源型別；整句確認 `default/test-rayjob` 這個 RayJob 已不存在。

## 6. assert.Empty(gotCluster.OwnerReferences) 必要嗎？

```go
assert.Empty(t, gotCluster.OwnerReferences)
```

這個測試的 cluster 原本沒有 owner reference，此斷言確認刪除流程後仍為空。

它不是這次 bug 的核心必要斷言：測試直接從已完成的 RayJob 開始，沒有經過建立 cluster 的路徑。不過 fake client 不會模擬 Kubernetes garbage collection，因此只確認 cluster 還存在，不足以發現意外建立的 owner reference。留著作為額外檢查合理。

不能推廣成所有既有 cluster 的 OwnerReferences 都必須為空；這裡是根據測試 fixture 的初始狀態做檢查。

## 7. Review 結論

現有測試已涵蓋 finalizer 清理，無須再要求補一份相同測試。可以 Approve 並留言：

> LGTM!

或補一句具體理由：

> LGTM! Moving the clusterSelector check allows DeleteSelf to run while preserving the shared RayCluster.

## 8. 單元測試與 manual test 的差別

跑單元測試用 `go test`：

```bash
cd ray-operator
go test ./controllers/ray -run '^TestReconcileRayJobClusterSelectorDeletion$' -v -count=1
```

`go run` 則是啟動 operator，用來搭配真實 Kubernetes cluster 做 manual test。

本次討論沒有在本地執行測試或啟動 operator。

## 9. Manual test

前提：

- kubectl 指向測試 cluster，已安裝對應版本的 CRD。
- 同 namespace 有 ready 的 RayCluster `processing`。
- 使用這支 PR 的 operator，啟用 RayJobDeletionPolicy。
- 若 cluster 內已有 operator，先暫停它，避免與本地 operator 同時處理；記下原本 replicas，以便還原。
- 下列 kubectl 指令使用目前 context 的 namespace；若需指定，統一加上 `-n <namespace>`。

### Terminal 1：啟動 operator

在實際 checkout 的 `ray-operator` 目錄執行，截圖中的指令整理如下：

```bash
go run main.go \
  --feature-gates=RayJobDeletionPolicy=true \
  --enable-leader-election=false \
  --use-kubernetes-proxy
```

- feature gate 啟用 deletion policy。
- 關閉 leader election 適用於這次本地啟動流程；它不會停掉 cluster 裡其他 operator。
- Kubernetes proxy 用於本地存取 cluster 內的 Ray dashboard，避免直接依賴本機解析 head Service DNS。

以上是本次提供的 manual test 指令，未在本地執行確認。

### Terminal 2：建立 RayJob 並觀察

```bash
kubectl get raycluster processing

# 可先記錄 UID，之後比較，確認是同一個 cluster。
kubectl get raycluster processing -o jsonpath='{.metadata.uid}{"\n"}'

kubectl apply -f - <<'YAML'
apiVersion: ray.io/v1
kind: RayJob
metadata:
  name: test-deleteself
spec:
  submissionMode: HTTPMode
  entrypoint: python -c 'print("hello")'
  clusterSelector:
    ray.io/cluster: processing
  deletionStrategy:
    deletionRules:
      - policy: DeleteSelf
        condition:
          jobStatus: SUCCEEDED
          ttlSeconds: 30
YAML

kubectl get rayjob test-deleteself -w
```

觀察 job 成功，等待 TTL 到期和 controller 處理刪除。完成時間不是固定 3 秒。

`-w` 不會因 RayJob 被刪除就自動結束；按 Ctrl+C 後再執行：

```bash
kubectl get rayjob test-deleteself
# 預期：NotFound

kubectl get raycluster processing
# 預期：仍存在且 ready

kubectl get raycluster processing -o jsonpath='{.metadata.uid}{"\n"}'
# 預期：與測試前相同
```

operator log 可輔助確認 `Executing deletion policy: DeleteSelf`。

### 為什麼不用 -l name=...？

`kubectl get ... -l name=...` 查的是 `metadata.labels.name`，不是 metadata.name。作者的 YAML 沒有這個 label，直接用資源名稱查更清楚，避免把查不到 label 誤判為物件已刪除。

### 測試後還原

先停止本地 operator，再將原本 operator deployment 恢復到測試前的 replicas。如果原本就是 1，且 deployment 名為 kuberay-operator：

```bash
kubectl scale deployment kuberay-operator --replicas=1 -n <operator-namespace>
```

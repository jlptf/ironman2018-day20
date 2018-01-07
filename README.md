## Day 20 - 常見問題與建議 (1)

### 本日共賞

* 錯誤的容器名稱或權限不足
* 驗證錯誤

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)


沒有人希望有錯誤發生！尤其是新手上路，更容易在錯誤發生後，因為無法修復而導致一連串挫折就放棄。而在 k8s 這樣龐大的系統下，新手發生錯誤的機會又特別高。因此，在接下來的幾天時間，我將列出 k8s 常見的錯誤與建議修正方法。

> 錯誤的發生往往是忽略一些小細節，通常 k8s 都會提示你錯誤是什麼

<br/>

#### 錯誤的容器名稱或權限不足

第一種常見的問題是當 k8s 嘗試建立 Pod 的時候找不到指定的映像檔 (image) 或者指到一個僅供私人存取的映像檔 (private image)，毫無疑問，肯定是無法順利建立 Pod。

> 極有可能是打錯字...

舉個例子，透過 run 指令建立一個 Pod 並指定一個不存在的映像檔 `james/idontexist:v1.0`

```bash
$ kubectl run doesnotexist --image=james/idontexist:v1.0
```

然後查看 Pod 狀態

```bash
$ kubectl get pods
NAME                            READY     STATUS             RESTARTS   AGE
doesnotexist-688bcb554f-gtj4z   0/1       ImagePullBackOff   0          8s
```

> 你可能會看到 ErrImagePull 與 ImagePullBackOff 兩種狀態，皆表示無法取得映像檔

這時候你會發現 Pod 的狀態不是 Running 表示 Pod 並沒有正常運作，接著查看詳細資料內 Event 的內容

```bash
$ kubectl describe pods doesnotexist-688bcb554f-gtj4z   <=== 名字要記得換掉喔
Events:
  Type     Reason                 Age                From               Message
  ----     ------                 ----               ----               -------
  Normal   Scheduled              40s                default-scheduler  Successfully assigned doesnotexist-688bcb554f-gtj4z to minikube
  Normal   SuccessfulMountVolume  40s                kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-ld9jt"
  Normal   Pulling                23s (x2 over 39s)  kubelet, minikube  pulling image "james/idontexist:v1.0"
  Warning  Failed                 18s (x2 over 35s)  kubelet, minikube  Failed to pull image "james/idontexist:v1.0": rpc error: code = Unknown desc = Error response from daemon: repository james/idontexist not found: does not exist or no pull access
  Warning  FailedSync             6s (x4 over 35s)   kubelet, minikube  Error syncing pod
  Normal   BackOff                6s (x2 over 34s)   kubelet, minikube  Back-off pulling image "james/idontexist:v1.0"

```

k8s 會提示你 `Failed to pull image "james/idontexist:v1.0"`，當你看到這個錯誤訊息就是 k8s 在告訴你它沒辦法取得 `james/idontexist:v1.0` 這個映像檔。

那麼問題來了，到底是什麼原因造成 k8s 無法取得映像檔？有以下幾種可能

1. 映像檔不存在
2. k8s 沒有權限取得映像檔 (pull image)
3. 映像檔存在，但是 tag 不正確

映像檔不存在有可能是打錯字造成的，如果想測試映像檔是否存在，可以用 docker 來測試，例如

```bash
$ docker pull james/idontexist:v1.0
```

> 如何安裝 docker 請參考 [docker 官網](https://www.docker.com/)

另一種情況是映像檔存在但是指定的 tag 不存在，這時候還可以把 tag 移除試試看。

```bash
$ docker pull james/idontexist
```

如果使用 `docker pull` 也無法抓取則很有可能映像檔真的不存在。但如果使用上述指令可以抓取但是 k8s 卻無法正確取得就有可能是權限不足的問題。

> 針對權限不足的狀況，可以利用 [Day 17 - Secret](https://ithelp.ithome.com.tw/articles/10193940) 提到的方法新增 Secret。

還有一種可能性，就是映像檔可能不是放在 [Dockerhub](https://hub.docker.com)。

由於 k8s 會將 [Dockerhub](https://hub.docker.com) 當作預設的映像檔儲存區 (repository registry)，因此，當你指定 `james/idontexist` 時，k8s 會認定你是使用 Dockerhub。所以，如果你是使用像 [AWS GCR](https://aws.amazon.com/tw/ecr/) 或 [Google Container Registry](https://cloud.google.com/container-registry/) 這些非 Dockerhub 的儲存區時，請務必將完整的 url 填入。例如 `gcr.io/james/idontexist`

> k8s 對於無權限存取或是映像檔不存在皆以 `ErrImagePull` 表示

<br/>

#### 驗證錯誤

還記得 [ Day 9 - yaml](https://ithelp.ithome.com.tw/articles/10193509) 討論過的 yaml 格式嗎？ 由於部署 k8s 物件需要撰寫 yaml 描述物件，常常一不小心就可能會打錯字或格式錯誤。另外一種常見的情況是 yaml 格式沒錯誤，但是指定的版本 (apiVersion) 並不支援某個設定或是放錯位置。撇除 yaml 格式錯誤的問題，我們來看一下底下的 error-obj.yaml

```
# error-obj.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: err-obj-nginx
spec:
    replicas: 3
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx  <=== ports 應該與這裡對齊
        ports:          <=== 放錯位置了
        - containerPort: 80
```

以 yaml 格式來說，上面的例子並沒有任何錯誤。但當我們部署到 k8s 中

```bash
$ kubectl apply -f error-obj.yaml
error: error validating "error-obj.yaml": error validating data: ValidationError(Deployment.spec.template.spec): unknown field "ports" in io.k8s.api.core.v1.PodSpec; if you choose to ignore these errors, turn validation off with --validate=false
```

`unknown field "ports"` 這邊會發現 `v1.PodSpec` 並不認得 ports 這個設定，其實不是沒有而是位置放錯了，正確寫法為

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: err-obj-nginx
spec:
    replicas: 3
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:    <=== 修正這裡
          - containerPort: 80
```

還有一種錯誤原因為指定的版本 (apiVersion) 並不支援某個設定，例如底下的 error-unknown.yaml

```
# error-unknown.yaml

---
apiVersion: v1
kind: Deployment   <=== v1 並無支援 Deployment 物件
metadata:
    name: err-unknown-nginx
spec:
    replicas: 3
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
```

就 yaml 格式上來看，error-unknown.yaml 沒有任何錯誤，部署到 k8s 後會看到以下錯誤

```
$ kubectl apply -f error-unknown.yaml
error: error validating "error-unknown.yaml": error validating data: unknown object type schema.GroupVersionKind{Group:"", Version:"v1", Kind:"Deployment"}; if you choose to ignore these errors, turn validation off with --validate=false
```

`unknown object type schema` 這邊 v1 並沒有 Deployment 這個物件，因此部署就會發生錯誤。

> 版本支援的物件請參考 [官方網站](https://kubernetes.io/docs/reference/) 的說明

*建議方案*

為避免上述錯誤，除了多加練習撰寫 yaml 之外，也可多多參考官方文件說明。另外 kubectl 有提供 `--dry-run` 供測試使用。如果要測試修正過後的 error-obj.yaml

```bash
$ kubectl create -f error.obj.yaml --dry-run
deployment "err-obj-nginx" created (dry run)
```

如果 yaml 存在錯誤就可以從這裡知道，如果沒有錯誤也不會真的部屬到 k8s 中。

> `kubectl --dry-run` 只有在能連到 k8s 下才能使用

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day20/](https://jlptf.github.io/ironman2018-day20/)
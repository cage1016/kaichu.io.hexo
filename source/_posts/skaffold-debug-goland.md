---
title: Skaffold debug goland
tags:
  - kubeneters
  - skaffold
  - debug
  - golang
  - vscode
  - GoLand
date: 2019-10-04 01:03:16
desc: skaffold v38.0 開始支持 Go container debugging support，非常大程度解決先前 remote debug golang 的痛苦
---


今年的 [Google Cloud Summit 台北場](https://inthecloud.withgoogle.com/summit-tpe-19/agenda.html) 於 9/24 在內湖萬豪酒店舉行，在主題演講中再次聽到 [Cloud Code](https://cloud.google.com/code) 的部份也看到比較有感覺的 Live demo

> Cloud Code 隨附的工具能協助您以輕鬆快速的方式編寫、部署及偵錯雲端原生的應用程式。這項產品提供 Visual Studio Code 和 IntelliJ 等 IDE 的擴充功能，方便您在 Kubernetes 上針對程式碼快速進行疊代、偵錯及部署等作業。
> &nbsp;
> Cloud Code 是基於 [GoogleContainerTools/skaffold: Easy and Repeatable Kubernetes Development](https://github.com/GoogleContainerTools/skaffold) 為基礎上再整合 Visual Studio Code 和 IntelliJ IDE, 讓開發者可以直接在 IDE 上就可以快速開發 Kubernetes 上的應用程式

基本上來說 Cloud Code 給 Visual Studio Code 和 IntelliJ IDE 的 plugin 就是基於 skaffold 打造的(如下圖)，當你在 Visual Studio Code 執行(cmd+shift+p) `Cloud Code: Continues Deploy` 的 log 就會知道

![](/img/skaffold-go-debug-image-4.png)

<!--more-->

## Cloud Code & debug

在談談 Skaffold debug k8s golang 的程式前，先來看看目前 Cloud Code debug 的部份支援幾種程式語言

_Visual Studio Code 中 Cloud code 可以新增的專案範本_

![](/img/skaffold-go-debug-image-5.png)

> 依照官方文件，現在 Cloud Code 有整合 debug 的部份有 `Node.js`、 `Python`、`Java`。 `Go` 目前不支援, 不過在 skaffold v38.0 的時候終於加進了. (Add Go container debugging support [#2306](https://github.com/GoogleContainerTools/skaffold/pull/2306))，所以就有了這一篇文章。至於本來 Cloud Code 就有整合的語言就依照官方的操作流程應該就可以正常運作，所以這邊就不在說明了

## Golang debug & skaffold

當我們開發 kubernetes 應用程式(golang)會有 remote debug 的需求，在 skaffold v38.0 版之前 remote debug golang 是需要比較複雜的方式(docker-compose or kubernetes))

**之前我們怎麼進行 remote debug golang**

1. 編譯 binary 時需要加入 `github.com/derekparker/delve/cmd/dlv`
2. 執行 binary 時需要帶入 `CMD ["/go/bin/dlv", "--listen=:2345", "--headless=true", "--api-version=2", "exec", "/exe"]`
3. docker-compose & kubernetes YAML 檔都需要加入額外的聲明(完整的 yaml 範例可以看 [這](https://github.com/cage1016/gokitconsul/blob/master/deployments/docker/docker-compose-debug.yaml#L57-L85))
    ```yaml
    ...
    security_opt:
      - apparmor:unconfined
    cap_add:
      - SYS_PTRACE
    ...
    ```
4. 需要 port forwarding dlv debug port 到本機
5. 透過 localhost port forwarding 就可以 attach 到 remote 的 golang 

步驟有一點麻煩

**當 skaffold v38.0 開始支援 container debugging support，事情就變簡單了**

1. remote debug 還是需要 `dlv`，不過 skaffold 透過 kubernetes 中的 initContainers 的方式來幫我們載入(Mount)相關的套件(dlv/dbg)，且會自動識別 runtime 來決定載入的套件，需 golang 的部份就在 `gcr.io/gcp-dev-tools/duct-tape/go`，(其他的 runtime 可由 https://github.com/GoogleContainerTools/container-debug-support/ 查詢)，並且會在 metadata 的 anootation 中加上 `debug.cloud.google.com/config: {"addsvc":{"dlv":56268,"runtime":"go"}}`
1. remote debug 還是需要 `dlv` 相關的啟動參數，skaffold 會自動幫我載入 
    ```yaml 
    Containers:
        addsvc:
            Container ID:  docker://493a405f1dc0ee462f4e5f269ae2fdb352e8468c186cbb26905c42e0bf5675cf
            Image:         cage1016/skaffold-debug-go-demo-addsvc:88f846061af2d34e8347a6325dcf48bb638f6baa50fd4599240fa5280054048e
            Image ID:      docker://sha256:88f846061af2d34e8347a6325dcf48bb638f6baa50fd4599240fa5280054048e
            Ports:         8020/TCP, 8021/TCP, 56268/TCP
            Host Ports:    0/TCP, 0/TCP, 0/TCP
            Command:
                /dbg/go/bin/dlv
                exec
                --headless
                --continue
                --accept-multiclient
                --listen=localhost:56268
                --api-version=2
                /exe
    ```
1. docker-compose & kubernetes YAML `不`需要再加註其他的聲明
1. Cloud Code 上跑 skaffold debug 會自動幫我們建立 port forwarding
1. Cloud Code 上進行 deubg 的設定，
    ```json
            {
            "type": "cloudcode",
            "language": "Go",
            "request": "attach",
            "debugPort": 56268,
            "localRoot": "${workspaceFolder}/go",
            "remoteRoot": "/go/",
            "name": "skaffold-debug-go-demo",
            "podSelector": {
                "app": "addsvc"
            }
        }
    ```

## demo

> repo: https://github.com/cage1016/skaffold-debug-goland-demo

![](/img/skaffold-debug-goland-demo.gif)

**操作步驟**

1. git clone 
```sh
git clone git@github.com:cage1016/skaffold-debug-goland-demo.git
```

1. skaffod debug
```sh
$ skaffold debug -p debug
Listing files to watch...
List generated in 81.41µs
Generating tags...
 - cage1016/skaffold-debug-go-demo-addsvc -> cage1016/skaffold-debug-go-demo-addsvc:latest
Tags generated in 130.733µs
Checking cache...
 - cage1016/skaffold-debug-go-demo-addsvc: Found
Cache check complete in 48.237684ms
Tags used in deployment:
 - Since images are not pushed, they can't be referenced by digest
   They are tagged and referenced by a unique ID instead
 - cage1016/skaffold-debug-go-demo-addsvc -> cage1016/skaffold-debug-go-demo-addsvc:88f846061af2d34e8347a6325dcf48bb638f6baa50fd4599240fa5280054048e
Starting deploy...
 - deployment.apps/addsvc created
 - service/addsvc created
Deploy complete in 655.821236ms
Watching for changes...
[addsvc-577bc59f76-n9w4w install-go-support] Installing runtime debugging support files in /dbg
[addsvc-577bc59f76-n9w4w install-go-support] Installation complete
[addsvc-577bc59f76-n9w4w addsvc] API server listening at: 127.0.0.1:56268
[addsvc-577bc59f76-n9w4w addsvc] level=info ts=2019-10-03T16:24:15.4025831Z caller=main.go:138 service=addsvc protocol=HTTP exposed=8020
[addsvc-577bc59f76-n9w4w addsvc] level=info ts=2019-10-03T16:24:15.4040995Z caller=main.go:151 service=addsvc protocol=GRPC protocol=GRPC exposed=8021
[addsvc-577bc59f76-n9w4w install-go-support] <Container was Terminated>
```

1. 設定中斷點

2. 啟動 Visual Studio Code 功能 (skaffold-debug-go-demo)
![](/img/skaffold-go-debug-image-6.png)

4. send request
```sh
$ make sum
curl -X "POST" "http://localhost:8020/sum" -H 'Content-Type: application/json; charset=utf-8' -d '{ "a": 3, "b": 34}'
{"rs":37,"err":null}

or

$ make concat
curl -X "POST" "http://localhost:8020/concat" -H 'Content-Type: application/json; charset=utf-8' -d '{ "a": "3", "b": "34"}'
{"rs":"334","err":null}
```

1. 這時候應該就可以正確跳至中斷點
![](/img/skaffold-go-debug-image-7.png)

**完整的 pod**

> 可以參照上面的說明來看

```yaml
$ kubectl describe po addsvc-577bc59f76-n9w4w
Name:           addsvc-577bc59f76-n9w4w
Namespace:      default
Node:           docker-for-desktop/192.168.65.3
Start Time:     Fri, 04 Oct 2019 00:24:11 +0800
Labels:         app=addsvc
                app.kubernetes.io/managed-by=skaffold-v0.39.0
                pod-template-hash=1336715932
                skaffold.dev/builder=local
                skaffold.dev/cleanup=true
                skaffold.dev/deployer=kubectl
                skaffold.dev/docker-api-version=1.39
                skaffold.dev/profiles=debug
                skaffold.dev/run-id=b77af1a5-0d52-4259-98d7-d1b09ed73c47
                skaffold.dev/tag-policy=sha256
                skaffold.dev/tail=true
Annotations:    debug.cloud.google.com/config: {"addsvc":{"dlv":56268,"runtime":"go"}}
Status:         Running
IP:             10.1.7.116
Controlled By:  ReplicaSet/addsvc-577bc59f76
Init Containers:
  install-go-support:
    Container ID:   docker://d7eb281035a72d6eb522860f6783eddaa146a3347c4b07ba41864dd5d1b1c5e9
    Image:          gcr.io/gcp-dev-tools/duct-tape/go
    Image ID:       docker-pullable://gcr.io/gcp-dev-tools/duct-tape/go@sha256:bacfc35dcd3b2c949f16290fbfcf82079a2b206deec2ee57e9606dfe7a913d71
    Port:           <none>
    Host Port:      <none>
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 04 Oct 2019 00:24:13 +0800
      Finished:     Fri, 04 Oct 2019 00:24:13 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /dbg from debugging-support-files (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-w97gc (ro)
Containers:
  addsvc:
    Container ID:  docker://493a405f1dc0ee462f4e5f269ae2fdb352e8468c186cbb26905c42e0bf5675cf
    Image:         cage1016/skaffold-debug-go-demo-addsvc:88f846061af2d34e8347a6325dcf48bb638f6baa50fd4599240fa5280054048e
    Image ID:      docker://sha256:88f846061af2d34e8347a6325dcf48bb638f6baa50fd4599240fa5280054048e
    Ports:         8020/TCP, 8021/TCP, 56268/TCP
    Host Ports:    0/TCP, 0/TCP, 0/TCP
    Command:
      /dbg/go/bin/dlv
      exec
      --headless
      --continue
      --accept-multiclient
      --listen=localhost:56268
      --api-version=2
      /exe
    State:          Running
      Started:      Fri, 04 Oct 2019 00:24:14 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      QS_ADDSVC_GRPC_PORT:  8021
      QS_ADDSVC_HTTP_PORT:  8020
      QS_ADDSVC_LOG_LEVEL:  info
    Mounts:
      /dbg from debugging-support-files (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-w97gc (ro)
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
Volumes:
  debugging-support-files:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  default-token-w97gc:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-w97gc
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason                 Age   From                         Message
  ----    ------                 ----  ----                         -------
  Normal  Scheduled              51s   default-scheduler            Successfully assigned addsvc-577bc59f76-n9w4w to docker-for-desktop
  Normal  SuccessfulMountVolume  51s   kubelet, docker-for-desktop  MountVolume.SetUp succeeded for volume "debugging-support-files"
  Normal  SuccessfulMountVolume  51s   kubelet, docker-for-desktop  MountVolume.SetUp succeeded for volume "default-token-w97gc"
  Normal  Pulling                50s   kubelet, docker-for-desktop  pulling image "gcr.io/gcp-dev-tools/duct-tape/go"
  Normal  Pulled                 49s   kubelet, docker-for-desktop  Successfully pulled image "gcr.io/gcp-dev-tools/duct-tape/go"
  Normal  Created                49s   kubelet, docker-for-desktop  Created container
  Normal  Started                49s   kubelet, docker-for-desktop  Started container
  Normal  Pulled                 48s   kubelet, docker-for-desktop  Container image "cage1016/skaffold-debug-go-demo-addsvc:88f846061af2d34e8347a6325dcf48bb638f6baa50fd4599240fa5280054048e" already present on machine
  Normal  Created                48s   kubelet, docker-for-desktop  Created container
  Normal  Started                48s   kubelet, docker-for-desktop  Started container
```

## Reference

1. [cage1016/skaffold-debug-go-demo: demo how to use skaffold remote debug k8s golang microservice](https://is.gd/KnwJi0)
2. [Debugging with Skaffold | Skaffold](https://skaffold.dev/docs/how-tos/debug/)
3. [Add Go container debugging support by briandealwis · Pull Request #2306 · GoogleContainerTools/skaffold](https://github.com/GoogleContainerTools/skaffold/pull/2306)
4. [GoogleContainerTools/container-debug-support: Language-runtime support files for debugging](https://github.com/GoogleContainerTools/container-debug-support/)
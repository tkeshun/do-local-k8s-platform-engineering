~~追記：このコマンドだめ↓~~ 結局行けるかもKind特有でPortをコントロールプレーンに設定する必要ある
```
cat <<EOF | kind create cluster --name dev --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    # nodeにラベルを付与
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  # ホストの 80 番ポートを Kind クラスターの 80 に接続（HTTP 用）。
  # ホストの 443 番ポートを Kind クラスターの 443 に接続（HTTPS 用）。
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
# ワーカーノードを3つ作成
- role: worker
- role: worker
- role: worker
EOF
```

必要なイメージのPull

cat ./platforms-on-k8s/chapter-2/kind-load.sh 
```
# docker imageの取得
docker pull bitnami/redis:7.0.11-debian-11-r12
docker pull bitnami/postgresql:15.3.0-debian-11-r17
docker pull bitnami/kafka:3.4.1-debian-11-r0
docker pull registry.k8s.io/ingress-nginx/controller:v1.8.1
docker pull registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407
docker pull salaboy/frontend-go-1739aa83b5e69d4ccb8a5615830ae66c:v1.0.0
docker pull salaboy/agenda-service-0967b907d9920c99918e2b91b91937b3:v1.0.0
docker pull salaboy/c4p-service-a3dc0474cbfa348afcdf47a8eee70ba9:v1.0.0
docker pull salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
# kindクラスターにイメーいjのロード、事前にイメージをロードすることで都度のPullを不要にしデプロイを高速化する
kind load docker-image -n dev bitnami/redis:7.0.11-debian-11-r12
kind load docker-image -n dev bitnami/postgresql:15.3.0-debian-11-r17
kind load docker-image -n dev bitnami/kafka:3.4.1-debian-11-r0
kind load docker-image -n dev registry.k8s.io/ingress-nginx/controller:v1.8.1
kind load docker-image -n dev registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407
kind load docker-image -n dev salaboy/frontend-go-1739aa83b5e69d4ccb8a5615830ae66c:v1.0.0
kind load docker-image -n dev salaboy/agenda-service-0967b907d9920c99918e2b91b91937b3:v1.0.0
kind load docker-image -n dev salaboy/c4p-service-a3dc0474cbfa348afcdf47a8eee70ba9:v1.0.0
kind load docker-image -n dev salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
```


↓以下リンクだめ  リンク切れ
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/release-1.8/deploy/static/provider/kind/deploy.yaml  

↓こちらでやった（https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md）  追記：これもそのままだとkindで使えない
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml  

新しいリンクだとnodeSelectorの設定が入ってない

コピペしてきて、nodeSelectorを追加  
ingress-ready: "true"をKind: Deploymentの下の方にあるnodeSelectorに追加  
applyしてもエラーになる
`<省略> strict decoding error: unknown field "spec.template.nodeSelector"`
一度deleteしてからapplyし直した
`kubectl delete deployment ingress-nginx-controller -n ingress-nginx`

taintがkindのcontrol-planeに設定されててデプロイできない→コントロールプレーンにはPodをスケジュールしないらしい。  
taintを外す必要があるらしい。→他のPodがスケジュールされても困るので、WorkNodeの一個にスケジュールされるようにする.  

一旦cluster消す。

kind delete cluster -n dev

ついでにファイル化しとく

ラベル一覧を取得
```
kubectl get nodes --show-labels
dev-worker          Ready    <none>          4m4s    v1.29.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress-ready=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=dev-worker,kubernetes.io/os=linux
```
ついてる  
kind loadから再びやる  
終わったらデプロイする  
できるまでみる  
```
kubectl -n ingress-nginx get pod -o wide -w
NAME                                        READY   STATUS      RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-htzb9        0/1     Completed   0          20s   10.244.2.2   dev-worker2   <none>           <none>
ingress-nginx-admission-patch-k8mcc         0/1     Completed   1          20s   10.244.1.2   dev-worker3   <none>           <none>
ingress-nginx-controller-57b4ccf946-ql9vv   0/1     Running     0          20s   10.244.3.2   dev-worker    <none>           <none>
ingress-nginx-controller-57b4ccf946-ql9vv   1/1     Running     0          28s   10.244.3.2   dev-worker    <none>           <none>
```

できた

helm のインストール
https://helm.sh/ja/docs/intro/install/

インストールと場所移動、確認
```
wget https://get.helm.sh/helm-v3.17.1-linux-amd64.tar.gz
tar -zxvf ./helm-v3.17.1-linux-amd64.tar.gz 
mv linux-amd64/helm /usr/local/bin/helm
which helm
```

つづき  
確認してからインストールする
```
helm show all oci://docker.io/salaboy/conference-app --version v1.0.0
helm install conference oci://docker.io/salaboy/conference-app --version v1.0.0
```
helm install時にChartに名前をつけてる

defaultのnamespaceにinstallされてた  
```
kubectl -n default get pods
NAME                                                           READY   STATUS    RESTARTS        AGE
conference-agenda-service-deployment-7787f7567d-k6gst          1/1     Running   2 (2m37s ago)   2m43s
conference-c4p-service-deployment-9c7d779c5-rr9k6              1/1     Running   2 (2m31s ago)   2m43s
conference-frontend-deployment-75f7dc4d8-l4tqk                 1/1     Running   2 (2m37s ago)   2m43s
conference-kafka-0                                             1/1     Running   0               2m42s
conference-notifications-service-deployment-86657dcd87-4z2v4   1/1     Running   2 (2m33s ago)   2m43s
conference-postgresql-0                                        1/1     Running   0               2m42s
conference-redis-master-0                                      1/1     Running   0               2m42s
```

疎通できない。調査。  
kindはcontrol planeしかポートマッピングできないっぽい？  
port mappingを移す  
IngressがLoadBalancerだったので、nodePortに変更  

おとしてきたIngress-nginxのマニフェストを変更  
前述のnodeSelector追加  
nodePortに変更、control-planeにデプロイされるように、tolentationを追加  

```
kubectl apply -f ingress-nginx.yaml 
kubectl -n ingress-nginx get pods -o wide
NAME                                        READY   STATUS      RESTARTS   AGE    IP           NODE                NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-ldxr8        0/1     Completed   0          6m4s   10.244.2.2   dev-worker2         <none>           <none>
ingress-nginx-admission-patch-49jj8         0/1     Completed   1          6m4s   10.244.1.2   dev-worker3         <none>           <none>
ingress-nginx-controller-7b8757c7bf-lfdqr   1/1     Running     0          6m4s   10.244.0.5   dev-control-plane   <none>           <none>
疎通確認
kubectl port-forward --namespace ingress-nginx service/ingress-nginx-controller 8080:80
```

`curl http://localhost`  
Ingressはないので、普通の疎通はできないはず→できた  
IngressなくてもIngress-nginx自体への疎通ならできるっぽい  

# ch4

ArgoCD

ArgoCD CLIのインストール
https://argo-cd.readthedocs.io/en/stable/cli_installation/
```
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

クラスタにArgoCDをinstall
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

forwardでArgoのUIにルーティング
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

UserNameはadmin、パスワードは`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`
で取得

staging環境を作る
```
kubectl create ns staging
```

ArgoのUIからアプリを作成する
https://github.com/salaboy/platforms-on-k8s/blob/main/chapter-4/README-ja.md#%E3%82%B9%E3%83%86%E3%83%BC%E3%82%B8%E3%83%B3%E3%82%B0%E7%92%B0%E5%A2%83%E7%94%A8%E3%81%AE%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AE%E3%82%BB%E3%83%83%E3%83%88%E3%82%A2%E3%83%83%E3%83%97

Ch2のやつ起動しっぱなしだとIngressでエラーになる  
`kubectl delete ingress conference-frontend-ingress`で消して、syncでOK

生えた

```
kubectl -n staging get ingress
NAME                                   CLASS   HOSTS   ADDRESS        PORTS   AGE
staging-environment-frontend-ingress   nginx   *       10.96.14.236   80      37s
```

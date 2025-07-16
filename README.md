# DockerとKubernetesをLinuxで動かしてみよう！

この読み物は、Linux（CentOS 7）でDockerとKubernetes（k8s）を動かす読み物です。Dockerで**Dockerfile**を手動で書いてコンテナを作り、k8sで**YAMLファイル**を自分で書いてアプリをデプロイします。
CentOS 7のサポートは終了しているので、あくまでも学習用途です。

## 目次
- [Dockerって何？](#dockerって何？)
- [Dockerfileで何ができる？](#dockerfileで何ができる？)
- [なぜKubernetesにDockerが必要なの？](#なぜkubernetesにdockerが必要なの？)
- [Kubernetesで何ができるの？](#kubernetesで何ができるの？)
- [必要なもの](#必要なもの)
- [インストール手順](#インストール手順)
  - [1. Dockerをインストール](#1-dockerをインストール)
  - [2. kubectlをインストール](#2-kubectlをインストール)
  - [3. Kindをインストール](#3-kindをインストール)
- [Dockerfileを手動で作ってみよう！](#dockerfileを手動で作ってみよう！)
- [Kubernetesを動かしてみよう！](#kubernetesを動かしてみよう！)
  - [1. Kubernetesクラスタを作る](#1-kubernetesクラスタを作る)
  - [2. YAMLでコンテナをデプロイ](#2-yamlでコンテナをデプロイ)
  - [3. アプリにアクセス](#3-アプリにアクセス)
- [片付け方](#片付け方)
- [トラブルシューティング](#トラブルシューティング)
- [Dockerとk8sで何ができる](#dockerとk8sで何ができる)

## Dockerって何？
Dockerは、アプリを「コンテナ」という軽い箱に詰めて動かすツールです。コンテナは、アプリとその実行環境（ライブラリや設定）を一緒にするので、どんなPCやサーバーでも同じように動きます。

**例**: ピザを宅配するイメージ。ピザ（アプリ）を箱（コンテナ）に入れて、どこでも同じ味（動作）が保証！

### なんでDockerがすごい？
- **軽い**：仮想マシン（VM）より少ないリソースで動く。
- **持ち運び簡単**：自分のPCで作ったコンテナをサーバーに持っていっても動く。
- **統一環境**：チーム全員が同じ環境で開発できる。

## Dockerfileで何ができる？
Dockerfileは、コンテナを作るための「レシピ」です。自分でスクリプトを書いて、必要なソフトや設定をコンテナに詰め込めます。例えば：
- **依存関係のダウンロード**：アプリに必要なパッケージ（例：Nginx、Python、PHPのComposerなど）をインストール。
- **カスタム設定**：アプリの設定ファイルや環境変数を指定。
- **プロビジョニング**：コンテナが動くのに必要な環境を自動で整える。

**例**: Nginxを動かすなら、DockerfileでNginxをダウンロードして設定。PHPアプリなら、Composerで依存パッケージを入れることも！

## なぜKubernetesにDockerが必要なの？
Kubernetes（k8s）は、たくさんのコンテナを管理する「司令塔」です。Dockerがコンテナを作る工場なら、k8sは「コンテナをどう動かすか」を管理します。

**Dockerとk8sの関係**：
- Dockerは、Dockerfileでコンテナを準備。
- k8sは、コンテナを複数のサーバーに配置したり、壊れたら作り直したり、アクセスが増えたら増やしたり。

**例**: Dockerがピザを焼くシェフなら、k8sはピザ屋の店長。店長は「どのピザをどのオーブンで焼くか」「お客さんが増えたらシェフを増やすか」を決める。

## Kubernetesで何ができるの？
k8sを使うと、以下が実現できます：
- **スケールアップ**：アクセスが増えたら、Pod（コンテナ）を自動で増やす（Horizontal Pod Autoscaler: HPA）。
- **自動復旧**：Podが壊れたら、自動で新しいPodを起動。
- **デプロイの簡単化**：YAMLファイルでアプリの構成を定義し、スムーズに公開。
- **ノードスケール**：Podが足りない時、サーバー（ノード）を増やす（Cluster Autoscaler）。

**k8sとAWS Auto Scalingの違い**：
- **AWS Auto Scaling**：
  - **対象**：EC2インスタンス（仮想マシン）を増減。
  - **特徴**：シンプルでAWSネイティブ。例：トラフィック増でEC2を2台→3台（数分）。仮想マシン単位なので粗い。
  - **イメージ**：ピザ屋のキッチン（EC2）を増やす。ピザ（アプリ）の細かい管理はしない。
- **k8s Autoscaling**：
  - **対象**：Pod（コンテナ）を増減（HPA）、必要ならノード（VM）を増減（Cluster Autoscaler）。
  - **特徴**：コンテナ単位で細かくスケール（数秒）。マルチクラウド対応。例：Nginx Podを3→5、足りなければEC2追加。
  - **イメージ**：ピザ（Pod）を増やし、キッチン（ノード）が足りないなら増やす。
- **VMの中にコンテナ**：
  - k8sは仮想マシン（例：EC2やKindのノード）の中でコンテナ（Pod）を動かす。例：EC2インスタンス1台にNginx Podを3つデプロイ。
  - HPAでPodを増やし（例：3→5）、VMのリソースが足りない場合、Cluster AutoscalerがAWS Auto Scalingを使ってEC2を追加。
  - **例**：ピザ屋のキッチン（EC2）でピザ（Pod）を焼き、忙しくなったらピザを増やし、キッチンが足りなければ新しいキッチンを建てる。
- **一緒に使う場合**：
  - AWS EKSでは、k8sがPodをスケール（HPA）、必要ならEC2をスケール（Cluster Autoscaler＋AWS Auto Scaling Groups）。ローカルのKindでは、DockerコンテナがVMの代わり。

**例**：トラフィックが増えた時、k8sはNginx Podを数秒で増やし、VMが足りないならEC2を数分で追加。AWS Auto Scalingだけだと、EC2増減だけでコンテナの細かい管理はできない。

## 必要なもの
- **OS**: CentOS 7（他のLinuxでも可、要コマンド調整）
- **ツール**:
  - **Docker**: コンテナを作る。
  - **Kind**: Docker上でk8s環境を作る。
  - **kubectl**: k8sを操作。

## インストール手順
作業は**ホームディレクトリ**（`/home/username/`）で進めます。`pwd`で常にディレクトリを確認！

### 1. Dockerをインストール
**作業ディレクトリ**: `/home/username`
```bash
# ディレクトリ確認
pwd
# 出力例: /home/username

sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum makecache fast
sudo yum install docker-ce
sudo systemctl start docker
sudo systemctl enable docker
sudo groupadd docker
sudo usermod -aG docker $USER
```
**注意**: ログアウトしてログインし直してください。
**確認**:
```bash
pwd
# 出力例: /home/username
```

### 2. kubectlをインストール
**作業ディレクトリ**: `/home/username`
```bash
pwd
# 出力例: /home/username

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
sudo yum install -y kubectl
```
**確認**:
```bash
pwd
# 出力例: /home/username
```

### 3. Kindをインストール
**作業ディレクトリ**: `/home/username`
```bash
pwd
# 出力例: /home/username

curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
**確認**:
```bash
pwd
# 出力例: /home/username
```

## Dockerfileを手動で作ってみよう！
Dockerfileを自分で書いて、Nginxコンテナを作ります。依存関係（Nginxパッケージ）をダウンロードします。

**手順**:
1. 作業用フォルダを作成。
   ```bash
   cd ~  # ホームディレクトリに移動
   pwd
   # 出力例: /home/username
   mkdir my-nginx
   cd my-nginx
   pwd
   # 出力例: /home/username/my-nginx
   ```
2. `Dockerfile`を作成（絶対パス：`/home/username/my-nginx/Dockerfile`）。
   ```bash
   nano /home/username/my-nginx/Dockerfile
   ```
   内容：
   ```dockerfile
   # ベースイメージ（Ubuntuを使用）
   FROM ubuntu:20.04

   # 必要なパッケージを更新＆Nginxをインストール
   RUN apt-get update && apt-get install -y nginx

   # Nginxの設定ファイル
   COPY nginx.conf /etc/nginx/nginx.conf

   # コンテナ起動時にNginxを動かす
   CMD ["nginx", "-g", "daemon off;"]
   ```
3. サンプル`nginx.conf`を作成（絶対パス：`/home/username/my-nginx/nginx.conf`）。
   ```bash
   nano /home/username/my-nginx/nginx.conf
   ```
   内容：
   ```nginx
   events {}
   http {
       server {
           listen 80;
           server_name localhost;
           location / {
               root /usr/share/nginx/html;
               index index.html;
           }
       }
   }
   ```
4. コンテナをビルド。
   ```bash
   pwd
   # 出力例: /home/username/my-nginx
   docker build -t my-nginx:1.0 .
   ```
5. コンテナをテスト実行。
   ```bash
   docker run -d -p 8080:80 my-nginx:1.0
   # ブラウザで http://localhost:8080 にアクセス
   ```

| 変更前 | 変更後 |
|--------|--------|
| **状態**: ディレクトリに何もない<br>`/home/username/my-nginx: [Empty]` | **状態**: Nginxコンテナが動く<br>`/home/username/my-nginx: [Dockerfile, nginx.conf]`<br>**コンテナ**: `[Docker] --> [Nginx Container:80]`<br>**説明**: DockerfileでNginxをダウンロード・設定し、コンテナ起動。ポート8080でアクセス可能。 |

**補足**: PHPアプリなら、Composerで依存関係を管理。例：
```dockerfile
FROM php:7.4-apache
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
COPY composer.json /var/www/html/
RUN cd /var/www/html && composer install
```

## Kubernetesを動かしてみよう！
Dockerfileで作ったコンテナをk8sで動かし、YAMLでデプロイを定義します。VM（例：EC2やKindのノード）内でコンテナを動かすイメージも掴みます。

### 1. Kubernetesクラスタを作る
**作業ディレクトリ**: `/home/username`
```bash
cd ~  # ホームディレクトリに戻る
pwd
# 出力例: /home/username
kind create cluster --name my-cluster
kubectl cluster-info
kubectl get pods --all-namespaces
```

| 変更前 | 変更後 |
|--------|--------|
| **状態**: ローカルPCに何もない<br>`/home/username: [Empty]` | **状態**: k8sクラスタが起動<br>`/home/username: [Kind Cluster]`<br>**説明**: KindがDocker内にk8sクラスタを作成。KindのノードはVMの代わり。 |

### 2. YAMLでコンテナをデプロイ
YAMLで「どんなコンテナをどう動かすか」を定義。VM（Kindのノード）内でコンテナ（Pod）が動きます。

**手順**:
1. `nginx-deployment.yaml`を作成（絶対パス：`/home/username/my-nginx/nginx-deployment.yaml`）。
   ```bash
   cd ~/my-nginx
   pwd
   # 出力例: /home/username/my-nginx
   nano /home/username/my-nginx/nginx-deployment.yaml
   ```
   内容：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: my-nginx:1.0
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx
   spec:
     selector:
       app: nginx
     ports:
     - port: 80
       targetPort: 80
       nodePort: 32000
     type: NodePort
   ```
2. KindにDockerイメージをアップロード。
   ```bash
   pwd
   # 出力例: /home/username/my-nginx
   kind load docker-image my-nginx:1.0 --name my-cluster
   ```
3. YAMLを適用。
   ```bash
   kubectl apply -f /home/username/my-nginx/nginx-deployment.yaml
   kubectl get deployments
   kubectl get pods
   kubectl get services
   ```

| 変更前 | 変更後 |
|--------|--------|
| **状態**: クラスタにアプリなし<br>`/home/username/my-nginx: [Dockerfile, nginx.conf]`<br>`[Kind Cluster]` | **状態**: Nginxポッドがデプロイ<br>`/home/username/my-nginx: [Dockerfile, nginx.conf, nginx-deployment.yaml]`<br>`[Kind Node (VM)] --> [Nginx Pod:80]`<br>**説明**: YAMLでNginx Podを定義。Kindのノード（VM代わり）内でPodが動き、NodePort（32000）でアクセス可能。 |

### 3. アプリにアクセス
**作業ディレクトリ**: `/home/username/my-nginx`
```bash
pwd
# 出力例: /home/username/my-nginx
kubectl get services nginx
# 例: ポート32000の場合、http://localhost:32000
```

| 変更前 | 変更後 |
|--------|--------|
| **状態**: 外部からアクセス不可<br>`[No Access]` | **状態**: Nginxページが見える<br>`[External Client] --> [NodePort:32000] --> [Nginx Pod:80]`<br>**説明**: NodePort経由でNginxのページが表示。 |

## 片付け方
**作業ディレクトリ**: `/home/username`
```bash
cd ~
pwd
# 出力例: /home/username
kind delete cluster --name my-cluster
```

| 変更前 | 変更後 |
|--------|--------|
| **状態**: k8sクラスタが稼働<br>`/home/username: [Kind Cluster, my-nginx/]`<br>`/home/username/my-nginx: [Dockerfile, nginx.conf, nginx-deployment.yaml]`<br>`[Kind Node (VM)] --> [Nginx Pod:80]` | **状態**: クラスタが消滅<br>`/home/username: [my-nginx/]`<br>`/home/username/my-nginx: [Dockerfile, nginx.conf, nginx-deployment.yaml]`<br>**説明**: クラスタ削除でk8sリソース（ノード、Pod）が消えるが、ファイルは残る。 |

## トラブルシューティング
- ポッドの状態: `kubectl describe pods [ポッド名]`
- サービスの状態: `kubectl describe services [サービス名]`
- ログ確認: `kubectl logs [ポッド名]`
- **ディレクトリ迷子**：`pwd`で現在地を確認。ホームディレクトリに戻るなら`cd ~`。

## Dockerとk8sで何ができる
- **カスタムアプリ**：Dockerfileで自分のアプリ（例：PHP、Go）をコンテナ化。
- **YAMLで管理**：k8sのYAMLでデプロイやスケーリングを自由に設定。
- **スケールアップ**：
  - Podを増やす：`kubectl scale deployment nginx --replicas=3`
  - ノードを増やす：Cluster AutoscalerがVM（EC2やKindノード）を追加。
- **VM＋コンテナ**：AWS EKSならEC2内でPodを動かし、必要ならEC2をスケール。KindならDockerコンテナがVMの代わり。


# Ansible

## 作るもののイメージ

- オフィスビルを管理し、ボイラーやビル空調設備などから温度・湿度・圧力・水漏れ・その他ビル管理の情報を吸い上げる
  - 簡単なシステムを構築
- データの吸い上げ
  - AWS Greengrassを使って構築してみる
  - Kubernetesを使って構築してみる
  - Amazon IoT Coreに格納する
  
![Test Image 3](/ansible/doc/story.drawio.png)

## 今回の構成

- VMは手動で用意
- Ansibleを用意
- Kubernetesを用意
- 管理サーバということで、KubernetesのコントロールプレーンとAnsibleのサーバは１台で賄う
  - もちろん別々のほうが望ましいが集約する
    - PC台数減らしたい
    - どちらも管理ソフト
- ゲートウェイを１台設置
- ゲートウェイにデータを送るダウンストリームを１台設置

## 検証用VMの準備

検証では下記OSのVMを３台用意した

|Name|OS|Version|Type|Product|User|Address|
|-|-|-|-|-|-|-|
|k3ssv|Ubuntu| 22.04|server|arm64|opeadmin|10.211.55.24|
|k3sgw|Ubuntu| 22.04|server|arm64|opeadmin|10.211.55.25|
|k3sds|Ubuntu| 22.04|server|arm64|opeadmin|10.211.55.26|
|k3sds2|Ubuntu| 22.04|server|arm64|opeadmin|10.211.55.27|

## システム構成

```mermaid
flowchart LR
    subgraph k3ssv
    k3sserver([k3s Server Control plane])
    ansible
    end
    subgraph k3sgw
    k3sagent([k3s Agent ])
    subgraph docker
      ggc([Greengrass Core])
    end
    end
    subgraph k3sds
    k3sagent2([k3s Agent ])
    subgraph docker2[docker]
      client([IoT Core Client])
    end
    end
    
    k3sserver -- Container Management --> 
    k3sagent -- Manage --> docker
    k3sserver -- Container Management --> k3sagent2
    k3sagent2 -- Manage --> docker2
    client == temperature ==> ggc
    ggc == temperature ==> cloud[IoT Core in the cloud]
    ansible -. Setup .-> k3sds
    ansible -. Setup .->  k3sgw
    ansible -. Setup .->  ansible
```

## セットアップ

- VM用意する
- Ansibleをセットアップ
  - 手作業で行う
- Ansibleコントロールノードで、SSH公開鍵設定を行う
  - AnsibleでSSH接続設定
- AWS IoT Greengrass イメージを構築する
- Ansibleを実行する
  - k3s serverをセットアップ
  - k3s agentをセットアップ（ゲートウェイ用）
  - k3s agentをセットアップ（ダウンストリーム用）
  - ゲートウェイにGreengrassを導入
  - ダウンストリームにMQTTクライアント導入

opeadminは権限昇格可能にしておく。

https://www.shellhacks.com/ansible-sudo-a-password-is-required/

```sh
$ sudo visudo
# And append a line as follows:
opeadmin  ALL=(ALL) NOPASSWD:ALL
```

### Parallelsを使ったVMの準備

- Parallels Desktop で Ubuntu 22.04 Server を構築する
  - [Ubuntuのインストーラをダウンロード](https://ubuntu.com/download/server/arm)
  - opeadmin/passwordmk
  - parallels tools のインストールを選択。
    - mount -r -t iso9660 /dev/cdrom /media
    - cd /media
    - sudo ./install
    - sudo reboot

### Azureを使ったVMの準備

省略

### EC2を使ったVMの準備（未定稿）

ansibleでVMを３台作成する、その方法は

### Vagrantを使ったVMの準備

省略

## k3s Server(k3ssv)

Ansibleをインストールする。

```sh
# OpenSSH : SSHPass を利用する
sudo apt -y install sshpass

# Pythonインストール
opeadmin@k3ssv:~$ python3 --version
Python 3.10.12

opeadmin@k3ssv:~$ sudo apt install python3.10-venv

opeadmin@k3ssv:~$ mkdir workspace
opeadmin@k3ssv:~$ cd workspace/
opeadmin@k3ssv:~/workspace$ git clone -b k3s https://github.com/emboss369/2023-k8s-Team.git
Cloning into '2023-k8s-Team'...
remote: Enumerating objects: 7, done.
remote: Counting objects: 100% (7/7), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 7 (delta 0), reused 4 (delta 0), pack-reused 0
Receiving objects: 100% (7/7), done.
opeadmin@k3ssv:~/workspace$ cd 2023-k8s-Team/ansible/

opeadmin@k3ssv:~/workspace/2023-k8s-Team/ansible$ python3 -m venv k3s
opeadmin@k3ssv:~/workspace/2023-k8s-Team/ansible$ source k3s/bin/activate
# 以降、k3s環境で作業する
(k3s) ansible$ pip install pip --upgrade
(k3s) ansible$ pip install ansible==9.0.1
(k3s) ansible$ pip install stormssh
```

## Greengrassの構築

### Ansibleコントロールノードで、SSH公開鍵設定を行う

```playbook_ssh_authorize.yaml``` を ```-k``` 付きで実行する。

それ以降は公開鍵認証が使用できるようにする。

note: 事前にssh接続してfingerprintでyesを押しておくこと。

```sh
# 鍵の作成と配布
(k3s) opeadmin@k3ssv:ansible$ ansible-playbook -i inventory.yaml playbook_ssh_authorize.yaml -k
# 接続確認
(k3s) opeadmin@k3ssv:ansible$ ssh k3sgw hostname
k3sgw ←これが表示されること
(k3s) opeadmin@k3ssv:ansible$ ssh k3sds hostname
k3sds ←これが表示されること
```

### AWS IoT Greengrass イメージを構築する

[Document](https://docs.aws.amazon.com/ja_jp/greengrass/v2/developerguide/build-greengrass-dockerfile.html)

イメージの構築コマンド例を以下に示す。コマンドの詳細は[ドキュメント](https://docs.docker.com/)を参照。

作成したイメージは、Dockerリポジトリに配置しておく。

イメージ作成はVM上でなくともよい。今回はMacを使用した。

- 前提条件
  - Docker DesktopがインストールされているPCであること
    - Docker 4.19.0 (106363)
  - 今回検証した環境は Mac M1
    - 上記以外の環境では未検証

```sh
# awsからチェックアウト
git clone https://github.com/aws-greengrass/aws-greengrass-docker.git
cd aws-greengrass-docker
# バージョン指定
git checkout refs/tags/v2.5.3
# docker pullでローカルに取得する
docker pull --platform linux/amd64 amazonlinux:2
docker pull --platform linux/arm64/v8 amazonlinux:2
# ビルド
sudo docker build --platform linux/amd64 -t "amd64/aws-iot-greengrass:nucleus-version" ./
sudo docker build --platform linux/arm64/v8 -t "arm64/aws-iot-greengrass:nucleus-version" ./
# ログイン
docker login
# 任意の名前に変更
docker tag 973de5ff9ae9 emboss369/greengrass:2.5.3-amd64
docker tag 17dfd4d2e357 emboss369/greengrass:2.5.3-arm64
# 各プラットフォーム分をUpする
docker push emboss369/greengrass:2.5.3-arm64
docker push emboss369/greengrass:2.5.3-amd64 
# (オプション)マルチプラットフォーム対応する
docker manifest create emboss369/greengrass:2.5.3 emboss369/greengrass:2.5.3-amd64 emboss369/greengrass:2.5.3-arm64
docker manifest annotate --arch amd64 emboss369/greengrass:2.5.3 emboss369/greengrass:2.5.3-amd64
docker manifest annotate --arch arm64 emboss369/greengrass:2.5.3 emboss369/greengrass:2.5.3-arm64
docker manifest push emboss369/greengrass:2.5.3
```

### ダウンストリーム側で動作させるPythonスクリプトをDocker化する

温度と湿度を送信するだけのPythonスクリプト。ダウンストリームで動作させる。

```sh
# イメージを作ってコンテナに格納する（1回のみでよい）
docker pull --platform linux/amd64 python:3.10-slim
sudo docker build --platform linux/amd64 -t "emboss369/simulated-temperature-sensor:2.31-amd64" ./

docker pull --platform linux/arm64/v8 docker.io/python:3.10-slim
sudo docker build --platform linux/arm64/v8 -t "emboss369/simulated-temperature-sensor:2.31-arm64" ./

### 各プラットフォーム分をUpする
docker push emboss369/simulated-temperature-sensor:2.31-amd64
docker push emboss369/simulated-temperature-sensor:2.31-arm64
### (オプション)マルチプラットフォーム対応する
docker manifest create emboss369/simulated-temperature-sensor:2.31 \
  emboss369/simulated-temperature-sensor:2.31-arm64 \
  emboss369/simulated-temperature-sensor:2.31-amd64
docker manifest annotate --arch amd64 emboss369/simulated-temperature-sensor:2.31 \
  emboss369/simulated-temperature-sensor:2.31-amd64
docker manifest annotate --arch arm64 emboss369/simulated-temperature-sensor:2.31 \
  emboss369/simulated-temperature-sensor:2.31-arm64
docker manifest push emboss369/simulated-temperature-sensor:2.31
```

### Ansibleの実行

（再実行する場合）すでに作成済みのリソースを削除する処理をansibleに入れる。今のところは手作業で削除してください。

- IoT Coreにログインし、作成済みのリソースを削除する
  - モノ
    - e.g.k3s_gg_core
    - e.g.k3s_iot_client
  - モノのグループ
    - e.g.k3s_gg_core_group
  - Greengrassデバイス
    - コアデバイス
      - e.g.k3s_gg_core
    - デプロイ
      - e.g.Deployment for k3s_gg_core
  - セキュリティ
    - ポリシー
      - e.g.k3s_iot_policy
- Greengrass Core端末にログインし、永続化されているデータを削除
  - ssh k3sgw
  - sudo rm -rR /home/opeadmin/greengrass

```sh
# フォルダ移動
cd ~/workspace/2023-k8s-Team/ansible
# AWS_ACCESS_KEY_ID,AWS_SECRET_ACCESS_KEYをexport
export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>

# 設定を行う
## サーバー接続先IPなどの設定
inventory.yaml
## 環境設定
### 全体環境設定
roles/common_settings/defaults/main.yaml
### デプロイの環境設定（デプロイ先）
roles/client_deployment/defaults/main.yaml
### デプロイの環境設定（デプロイ先）
roles/greengrass_deployment/defaults/main.yaml

### 上記はcommonに集約されました
roles/common_settings/defaults/main.yaml

# Ansible実行
(k3s) opeadmin@k3ssv:ansible$ ansible-playbook -i inventory.yaml playbook_k3s.yaml --private-key ~/.ssh/id_rsa_ansible

unreachable=0    failed=0 ← 最後のPLAY RECAPにこれが含まれていること
```

### 手動でGreengrass CoreをDeployする場合

```sh
# 対象端末ログイン
export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
### ユーザ関係
cd 
sudo adduser --system ggc_user
sudo addgroup --system ggc_group
sudo useradd -s /bin/bash -m opeadmin
sudo passwd opeadmin
sudo usermod -aG ggc_group opeadmin

# Credensialの作成
sudo mkdir /home/opeadmin/greengrass-v2-credentials

### sudoからEOFまでが1つのコマンド
sudo tee /home/opeadmin/greengrass-v2-credentials/credentials <<EOF
[default]
aws_access_key_id = ${AWS_ACCESS_KEY_ID}
aws_secret_access_key = ${AWS_SECRET_ACCESS_KEY}
EOF

# log用ディレクトリの作成
sudo mkdir -p /home/opeadmin/greengrass/v2/logs
sudo chown -R opeadmin:ggc_group /home/opeadmin/
### 作成したファイルの確認
# Credensialの確認
sudo cat /home/opeadmin/greengrass-v2-credentials/credentials
# logディレクトリの確認
sudo ls -la /home/opeadmin/greengrass/v2/logs
cd ~/2023-k8s-Team/
cat greengrass-v2-deployment.yaml

# マスターログイン
cd 
git clone -b k3s https://github.com/emboss369/2023-k8s-Team.git
cd 2023-k8s-Team

cp ansible/roles/greengrass_deployment/defaults/main.yaml ansible/roles/greengrass_deployment/defaults/main_k8s.yaml
vim ansible/roles/greengrass_deployment/defaults/main_k8s.yaml
### ---- 編集例（環境に合わせて）
GGC_ROOT_PATH: /greengrass/v2
AWS_REGION: ap-northeast-1
THING_NAME: k8s_gg_core
THING_GROUP_NAME: k8s_gg_core_group
TES_ROLE_NAME: k8s_TokenExchangeRole
TES_ROLE_ALIAS_NAME: k8s_TokenExchangeRoleAlias
COMPONENT_DEFAULT_USER: ggc_user:ggc_group
DEPLOY_DEV_TOOLS: false
CONTAINER_PORT: 8883
CREDENTIL_PATH: /home/opeadmin/greengrass-v2-credentials/
TARGET_NODE_NAME: ip-10-0-35-104
### ---

cd ansible/roles/greengrass_deployment/templates/
sed 's/path.*v2/path: \/home\/opeadmin\/greengrass\/v2/g' \
greengrass-v2-deployment.j2 > greengrass-v2-deployment-k8s.j2

cd ~/2023-k8s-Team/

### catからEOFまでが1つのコマンド
cat <<EOF > likes_ansible_script.py
from jinja2 import Template, Environment, FileSystemLoader
import sys, yaml

env = Environment(loader=FileSystemLoader('./ansible/roles/greengrass_deployment/templates/', encoding='utf8'))
tmpl = env.get_template('greengrass-v2-deployment-k8s.j2')

with open('./ansible/roles/greengrass_deployment/defaults/main_k8s.yaml') as import_yml:
  sys.stdout.write(tmpl.render(yaml.load(import_yml)))
EOF

python3 likes_ansible_script.py > greengrass-v2-deployment.yaml

# k8sへのデプロイ
kubectl apply -f greengrass-v2-deployment.yaml

## create iot clinet(thing)
# ダウンストリーム端末での作業
## 任意の場所に証明書を置く
CERTS_PATH=/home/opeadmin/certs
THING_NAME=k8s_gg_core
IOT_THING_NAME=k8s_iot_client
IOT_POLICY_NAME=k8s_iot_policy
AWS_REGION=ap-northeast-1

mkdir $CERTS_PATH
cd $CERTS_PATH
aws iot create-thing --thing-name $IOT_THING_NAME
aws iot create-keys-and-certificate --set-as-active > cert.json
cat cert.json | jq .keyPair.PrivateKey -r > private.pem.key
cat private.pem.key
jq .certificateId ./cert.json > ./certificate-id.txt
cat certificate-id.txt
CERTIFICATED_ID=`cat ./certificate-id.txt | sed 's/"//g'`
echo $CERTIFICATED_ID
aws iot describe-certificate --certificate-id $CERTIFICATED_ID --output text  --query certificateDescription.certificatePem > device.pem.crt
openssl x509 -text < ./device.pem.crt
# マスター端末ログイン
cd ~/2023-k8s-Team/ansible
# - name: Copy policy templte
cp roles/iot/templates/k3s_iot_policy.j2 k8s_policy.json
# - name: Get IoT Policies
aws iot list-policies | jq '.policies[] | .policyName'
# - name: Create IoT Policy
aws iot create-policy --policy-name $IOT_POLICY_NAME --policy-document file://k8s_policy.json
# - name: Get IoT Policy
aws iot get-policy --policy-name $IOT_POLICY_NAME > $OT_POLICY_NAME.json
# ダウンストリーム端末での作業
# opeadminユーザでの作業
cd
# - name: Attach a IoT policy to a certificate
CERTARN=`jq .certificateArn ./certs/cert.json  | sed 's/"//g'`
aws iot attach-principal-policy --principal $CERTARN --policy-name $IOT_POLICY_NAME
aws iot list-principal-policies --principal $CERTARN
aws iot attach-thing-principal --thing-name $IOT_THING_NAME --principal $CERTARN
# - name: Show things principals
aws iot list-thing-principals --thing-name $IOT_THING_NAME
# - name: Show endpoint
aws iot describe-endpoint --region $AWS_REGION
# - name: Associate client device with core device
aws greengrassv2 batch-associate-client-device-with-core-device \
  --core-device-thing-name $THING_NAME \
  --entries thingName=$IOT_THING_NAME
# マスター端末ログイン
cd ~/2023-k8s-Team/ansible
# - name: Get IoT Core thingArn from describe-thing
THING_ARN=`aws iot describe-thing --thing-name $THING_NAME | jq .thingArn | sed 's/"//g'`
echo $THING_ARN　<-控えておく
# - name: Copy deployment templete
cp roles/iot/templates/deplyment.j2 cli-deployment.json
vim cli-deployment.json
{{ thingArn.stdout }}を控えておいた値に書き換え
{{ THING_NAME }}をTHING_NAMEで置き換え
# - name: Create Deplyment for greengrass core thing
aws greengrassv2 create-deployment \
  --cli-input-json file://cli-deployment.json
# ダウンストリーム端末での作業
# opeadminで作業
CERTS_PATH=/home/opeadmin/certs
cd $CERTS_PATH
wget https://www.amazontrust.com/repository/AmazonRootCA1.pem -O AmazonRootCA1.pem
# マスター端末ログイン
cd ~/2023-k8s-Team/ansible
# - name: IoT Client Application deployment file
cp roles/client_deployment/templates/client-deployment.j2 client-deployment.yaml
vim client-deployment.yaml
{{ と }} で囲まれた箇所を、すべて手動で置き換える。
{{ MQTT_TOPIC }}は今回のテストで使うdata/IoTTask-IoTCoreMonoを設定。
# - name: Apply IoT Client Application Pod
kubectl apply -f client-deployment.yaml
watch kubectl -n greengrass get pod -o wide
```

```sh
## タグ確認手順
kubectl get node
ip-10-0-11-46    Ready    control-plane   24d   v1.27.6
ip-10-0-22-233   Ready    <none>          24d   v1.27.6
ip-10-0-35-104   Ready    <none>          24d   v1.27.6
ip-10-0-43-50    Ready    <none>          24d   v1.27.6

kubectl describe nodes ip-10-0-22-233 | grep location
                    location=edge1 (k8s-worker-01)
kubectl describe nodes ip-10-0-43-50 | grep location
                    location=edge2
```

## kubernetesあるある、Deploymentをdeleteしたい

kubectl -n greengrass delete deployment client-deployment

kubectl -n greengrass get pod -o wide

kubectl apply -f client-deployment.yaml

### Dockerあるある、起動しない時の調査方法

```sh
tty: true
```

```sh
kubectl exec -n greengrass -it client-deployment-b4d794598-t8w6d /bin/bash
```

```sh
python3 -u SimulatedTemperatureSensor.py \
  --thing_name $THING_NAME \
  --topic $MQTT_TOPIC \
  --ca_file $CA_FILE \
  --cert $PEM_CRT \
  --key $PEM_KEY \
  --region $REGION \
  --verbosity ${VERBOS} \
  --max_pub_ops $MAX_PUB_OPS
```

### 以下、作業メモ領域

### IoT モノを作成

```sh
# Ubuntu 上に Node.js + npm の環境を構築する。

$ sudo apt update
$ sudo apt install nodejs npm
$ sudo npm -g install n
$ sudo n stable
# $ sudo apt purge nodejs npm
# $ sudo apt autoremove

# 一度シェルを再起動したのち、バージョンを確認する。

opeadmin@k3ssv:cdk_app$ node -v
v20.10.0
opeadmin@k3ssv:cdk_app$ npm -v
10.2.3

# https://aws.amazon.com/jp/getting-started/guides/setup-cdk/module-two/
# AWS CDK のインストール
opeadmin@k3ssv:cdk_app$ sudo npm install -g aws-cdk
opeadmin@k3ssv:cdk_app$ cdk --version
2.111.0 (build 2ccb59e)

# 結局、セットアップしたもののCDKはつかわずじまいだった。スキップ可能

# https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html
# AWS CLIのインストール(armの場合)

cd
opeadmin@k3ssv:~$ url "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
opeadmin@k3ssv:~$ unzip awscliv2.zip
opeadmin@k3ssv:~$ sudo ./aws/install

# AWS CLIのインストール(amdの場合)
opeadmin@k3ssv:~$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
opeadmin@k3ssv:~$ sudo apt install zip unzip
unzip awscliv2.zip
sudo ./aws/install

# AWS CLIのアンインストール
sudo rm /usr/local/bin/aws
sudo rm /usr/local/bin/aws_completer
sudo rm -rf /usr/local/aws-cli
sudo rm -rf ~/.aws/


# aws configure しておく

# aws cdkアプリケーションを作成する（1回だけ、作成済みのをGithubにUpするのでCloneするだけ。）
cd ~/workspace/2023-k8s-Team/cdk_app/
mkdir cdk-k3s-iot
cd cdk-k3s-iot
cdk init app --language typescript
# 結局 aws cdkは使わなかった。

# https://docs.aws.amazon.com/cli/latest/reference/iot/create-thing.html

aws iot delete-thing --thing-name k3s_iot_client
aws iot create-thing --thing-name k3s_iot_client

(k3s) opeadmin@k3ssv:cdk_app$ aws iot delete-thing --thing-name k3s_iot_client
(k3s) opeadmin@k3ssv:cdk_app$ aws iot create-thing --thing-name k3s_iot_client
{
    "thingName": "k3s_iot_client",
    "thingArn": "arn:aws:iot:ap-southeast-2:630878272277:thing/k3s_iot_client",
    "thingId": "56b25a01-b42e-4d08-914b-f97b3d904b1a"
}

cd ~/workspace/2023-k8s-Team/ansible
sudo apt install -y jq
mkdir cert
aws iot create-keys-and-certificate --set-as-active > ./cert/cert.json
cat ./cert/cert.json | jq .keyPair.PrivateKey -r > ./cert/private.pem.key
jq .certificateId ./cert/cert.json > ./cert/certificate-id.txt
CERTIFICATED_ID=`cat ./cert/certificate-id.txt | sed 's/"//g'`
aws iot describe-certificate --certificate-id $CERTIFICATED_ID --output text \
  --query certificateDescription.certificatePem > ./cert/cert.pem

# 確認
openssl  x509 -text < ./cert/cert.pem

# テンプレートの確認



# ルート証明書はクライアント上に必要。
# wget https://www.amazontrust.com/repository/AmazonRootCA1.pem -O AmazonRootCA1.pem

次はここから。

# クライアントデバイスとの関連付けを管理する (AWS CLI)
https://docs.aws.amazon.com/ja_jp/greengrass/v2/developerguide/associate-client-devices.html#manage-client-device-associations-cli

ssh k3sds
mkdir ~/certs
exit <- ansibleサーバに戻る

# certsのコピー svからdsへ。
(k3s) opeadmin@k3ssv:ansible$ scp cert/device.pem.crt k3sds:~/certs/device.pem.crt
device.pem.crt     100% 1225     2.9MB/s   00:00    
(k3s) opeadmin@k3ssv:ansible$ scp cert/private.pem.key k3sds:~/certs/private.pem.key
private.pem.key    100% 1676     4.1MB/s   00:00  

ssh k3sds
# ルートCA証明書のダウンロード
$ wget https://www.amazontrust.com/repository/AmazonRootCA1.pem -O certs/AmazonRootCA1.pem

# ダウンストリームからゲートウェイを経由し、IoT Coreにデータを通してみる（Dockerなしで）
git clone https://github.com/aws/aws-iot-device-sdk-python-v2.git
# AWS IoT Device SDK v2 for Python をインストールします。
sudo apt install -y  python3-pip
# AWS IoT Device SDK v2 for Python のサンプルフォルダに移動します。
cd aws-iot-device-sdk-python-v2/samples
# Hello World送ります
python3 basic_discovery.py \
  --thing_name k3s_iot_client \
  --topic 'clients/k3s_iot_client/hello/world' \
  --message 'Hello World!' \
  --ca_file ~/certs/AmazonRootCA1.pem \
  --cert ~/certs/device.pem.crt \
  --key ~/certs/private.pem.key \
  --region ap-southeast-2 \
  --verbosity Warn
  --max_pub_ops 36000

```


[クライアントデバイスとの関連付けを管理する (AWS CLI)](https://docs.aws.amazon.com/ja_jp/greengrass/v2/developerguide/associate-client-devices.html#manage-client-device-associations-cli)

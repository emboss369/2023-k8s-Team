# Ansible

## 今回の構成

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
    
    k3sserver --- k3sagent
    k3sserver --- k3sagent2
    client --> ggc
    ggc --> cloud[IoT Core in the cloud]
```

## 検証用VMの準備

検証では下記OSのVMを３台用意した

|OS|Version|Type|Product|user|
|-|-|-|-|
|Ubuntu| 22.04|server|arm64|opeadmin|

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

## サーバー、エージェントのインストール


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

### Ansibleの実行

export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>

(k3s) opeadmin@k3ssv:ansible$ ansible-playbook -i inventory.yaml playbook_k3s_server.yaml --private-key ~/.ssh/id_rsa_ansible


unreachable=0    failed=0 ← 最後のPLAY RECAPにこれが含まれていること

### AWS IoT Greengrass イメージを構築する

[Document](https://docs.aws.amazon.com/ja_jp/greengrass/v2/developerguide/build-greengrass-dockerfile.html)

イメージの構築コマンド例を以下に示す。コマンドの詳細は[ドキュメント](https://docs.docker.com/)を参照。

```sh
git clone https://github.com/aws-greengrass/aws-greengrass-docker.git
cd aws-greengrass-docker
git checkout refs/tags/v2.5.3

docker pull --platform linux/amd64 amazonlinux:2
docker pull --platform linux/arm64/v8 amazonlinux:2

sudo docker build --platform linux/amd64 -t "amd64/aws-iot-greengrass:nucleus-version" ./
sudo docker build --platform linux/arm64/v8 -t "arm64/aws-iot-greengrass:nucleus-version" ./

docker login
docker tag 973de5ff9ae9 emboss369/amd64/aws-iot-greengrass:2.5.3
docker tag 17dfd4d2e357 emboss369/arm64/aws-iot-greengrass:2.5.3
docker manifest annotate --arch amd64 emboss369/greengrass:2.5.3 emboss369/greengrass:2.5.3-amd64
docker manifest annotate --arch arm64 emboss369/greengrass:2.5.3 emboss369/greengrass:2.5.3-arm64
docker manifest push emboss369/greengrass:2.5.3
```


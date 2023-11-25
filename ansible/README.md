# Ansible

## k3s Server(k3ssv) 

- Ansibleのインストール（手作業）
  - [Ubuntuのインストーラをダウンロード](https://ubuntu.com/download/server/arm)
  - opeadmin/passwordmk
  - parallels tools のインストール　
    - mount -r -t iso9660 /dev/cdrom /media
    - cd /media
    - sudo ./install
  
```sh
opeadmin@k3ssv:~$ python3 --version
Python 3.10.12

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

```
## High Available な環境構築
　これまで StackStorm の使い方や機能拡張の方法など、どちらかというと開発寄りな内容について解説してきましたが、ここからは StackStorm の運用のポイントについて解説します。  
　本稿の冒頭で StackStorm が Scalable で High Available なアーキテクチャであることを説明しました。ここでは、冗長構成な StackStorm における運用の注意点について簡単に解説します。  

### 冗長構成な StackStorm の構築

　ここで冗長構成な StackStorm を構築します。  
　[StackStorm のドキュメント](https://docs.stackstorm.com/reference/ha.html#reference-ha-setup) に、以下の構成での構築方法が詳細に解説されていますが、ここでは簡単のために別の方法を紹介します。  

![StackStorm HA reference deployment](https://docs.stackstorm.com/_images/st2-deployment-multi-node.png)
(出典：[High availability deployment | StackStorm 2.0.1 documentation](https://docs.stackstorm.com/reference/ha.html))

　ここでは、最初に構築したノードをプライマリノードとし、これから構築するノードをセカンダリノードとする環境を構築します。図にすると以下のようになります。  

![StackStorm 冗長構成](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/ha-architecture.png)

　上記の構成で StackStorm の各サービスは冗長化しますが「プライマリノードが単一障害点になるじゃないか！」と思う鋭い読者もいるかもしれません。  
　確かに現状はその通りですが、[MongoDB のレプリケーション](https://docs.mongodb.com/manual/replication/) や [RabbitMQ の HA queue](https://www.rabbitmq.com/ha.html) を利用することで、完全に単一障害点を無くす構成の StackStorm 環境を構築することもできます。これらの解説については本稿の趣旨を超えるため、ここでは割愛し上記の構成で話を進めます。  

### プライマリノードの設定変更
　冗長構成を構築するに際し、プライマリノードにおいて、セカンダリノードからの MongoDB 及び RabbitMQ へのアクセスを許可するための設定と NFS の設定を行います。  

　まずは MongoDB にセカンダリノードからアクセスできるよう設定ファイル (/etc/mongodb.conf) ファイルを以下の通り修正し、サービスを再起動させます。  

```diff
--- etc/mongod.conf.orig	2016-11-28 07:18:22.490265432 +0000
+++ /etc/mongod.conf	2016-11-28 07:18:44.938264718 +0000
@@ -21,7 +21,7 @@
 # network interfaces
 net:
   port: 27017
-  bindIp: 127.0.0.1
+  bindIp: 192.168.0.100
 
 
 #processManagement:
```
```sh
vagrant@st2-node:~$ sudo service mongod restart
mongod stop/waiting
mongod start/running, process 21613
vagrant@st2-node:~$ 
```

　これに伴い StackStorm の設定ファイル `/etc/st2/st2.conf` を以下の通り修正し、サービスを再起動させます。  

```diff
--- etc/st2/st2.conf.orig	2016-11-28 07:08:40.748655911 +0000
+++ /etc/st2/st2.conf	2016-11-28 10:12:46.297432991 +0000
@@ -81,3 +81,6 @@
 
 [keyvalue]
 encryption_key_path = /etc/st2/keys/datastore_key.json
+
+[database]
+host = 192.168.0.100
```
```sh
vagrant@st2-node:~$ sudo st2ctl restart
```

　続いて RabbitMQ にセカンダリノード用のアカウントを作成します。これまでのマスターノードは、デフォルトで作成される `guest` アカウントでログインしていましたが、[`guest` アカウントはローカルホストからしかアクセス出来ない](https://www.rabbitmq.com/access-control.html) ため、ここで別アカウント `st2` を作成し、権限の付与を行います。  

```
vagrant@st2-node:~$ sudo rabbitmqctl add_user st2 passwd
Creating user "st2" ...
...done.
vagrant@st2-node:~$ sudo rabbitmqctl set_permissions st2 ".*" ".*" ".*"
Setting permissions for user "st2" in vhost "/" ...
...done.
vagrant@st2-node:~$ 
```

　最後に StackStorm で冗長構成を組む場合、コンテンツディレクトリ (`/opt/stackstorm/` ディレクトリ配下の `packs` と `virtualenvs` の２つ) を共有させる必要があります。ここではプライマリノードに NFS サーバを構築し、セカンダリノードからコンテンツディレクトリがマウントできるようにします。  

　まずは NFS サーバをパッケージからインストールします。  
```
vagrant@st2-node:~$ sudo apt-get install nfs-kernel-server
```

　続いて、設定ファイル `/etc/exports` にセカンダリノードに対して公開するディレクトリを指定します。以下に、追加した設定を示します。  
```diff
--- etc/exports.orig	2016-11-28 07:44:03.014216487 +0000
+++ /etc/exports	2016-11-28 07:48:33.258207901 +0000
@@ -8,3 +8,5 @@
 # /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
 # /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
 #
+/opt/stackstorm/packs         192.168.0.101(rw,sync,no_subtree_check,anonuid=0,anongid=999)
+/opt/stackstorm/virtualenvs   192.168.0.101(rw,sync,no_subtree_check,anonuid=0,anongid=999)
```
　オールインワンインストールをした場合、コンテンツディレクトリの所有者（ユーザ・グループ）は root/st2packs に設定されます。マウント先でも同様に扱えるようにするため `anonuid` オプションに root の uid を設定し、`anongid` オプションに st2packs の gid を設定します。尚 st2packs の gid は以下の方法で確認できます。  

```sh
vagrant@st2-node:~$ grep 'st2packs' /etc/group
st2packs:x:999:st2
vagrant@st2-node:~$ 
```

　最後に NFS サーバを再起動させれば完了です。  
```
vagrant@st2-node:~$ sudo service nfs-kernel-server restart
 * Stopping NFS kernel daemon
   ...done.
 * Unexporting directories for NFS kernel daemon...
   ...done.
 * Exporting directories for NFS kernel daemon...
   ...done.
 * Starting NFS kernel daemon
   ...done.
vagrant@st2-node:~$ 
```

### セカンダリノードの構築
　続いて、セカンダリノードの構築を行います。  
　まず、これまでと同じ方法でセカンダリノードを作成し、StackStorm をインストールします。その際 Vagrantfile に設定したプライベートネットワークの IP アドレスを最初のノードに設定したものと違う値を設定してください。ここでは以下の通り Vagrantfile を修正して、[StackStorm のインストール](...) で解説した手順でインストールします。  

```diff
--- vagrantfile.orig	2016-11-28 16:38:22.000000000 +0900
+++ vagrantfile	2016-11-28 16:38:30.000000000 +0900
@@ -27,6 +27,8 @@
   # create a private network, which allows host-only access to the machine
   # using a specific ip.
   # config.vm.network "private_network", ip: "192.168.33.10"
+  config.vm.hostname = "st2-secondary"
+  config.vm.network "private_network", ip: "192.168.0.101"
 
   # create a public network, which generally matched to bridged network.
   # bridged networks make the machine appear as another physical device on
@@ -50,6 +52,10 @@
   #   # customize the amount of memory on the vm:
   #   vb.memory = "1024"
   # end
+  config.vm.provider "virtualbox" do |vb|
+    vb.cpus = 2
+    vb.memory = "2048"
+  end
   #
   # view the documentation for the provider you are using for more
   # information on available options.
```

　StackStorm がインストールされたら、DB・MQ の接続先をプライマリノードに変更するために StackStorm の設定ファイル `/etc/st2/st2.conf` を以下の通り修正します。  

```diff
--- etc/st2/st2.conf.orig	2016-11-28 09:33:07.591549477 +0000
+++ /etc/st2/st2.conf	2016-11-28 09:53:40.659583816 +0000
@@ -73,7 +73,7 @@
 ssh_key_file = /home/stanley/.ssh/stanley_rsa
 
 [messaging]
-url = amqp://guest:guest@127.0.0.1:5672/
+url = amqp://st2:passwd@192.168.0.100:5672/
 
 [ssh_runner]
 remote_dir = /tmp
@@ -81,3 +81,6 @@
 
 [keyvalue]
 encryption_key_path = /etc/st2/keys/datastore_key.json
+
+[database]
+host = 192.168.0.100
```

　続いて NFS でサーバのプライマリノードのコンテンツディレクトリをマウントし、実行環境及び静的データをプライマリノードと共有します。`/etc/fstab` を以下の通り修正し、それぞれのマウントポイントで指定したディレクトリをマウントをします。  

```diff
--- etc/fstab.orig	2016-11-28 09:55:20.319586592 +0000
+++ /etc/fstab	2016-11-28 09:56:58.639589330 +0000
@@ -1 +1,4 @@
 LABEL=cloudimg-rootfs	/	 ext4	defaults	0 0
+
+192.168.0.100:/opt/stackstorm/packs /opt/stackstorm/packs nfs defaults 0 0
+192.168.0.100:/opt/stackstorm/virtualenvs /opt/stackstorm/virtualenvs nfs defaults 0 0
```
```sh
vagrant@st2-secondary:~$ sudo mount /opt/stackstorm/packs
vagrant@st2-secondary:~$ sudo mount /opt/stackstorm/virtualenvs
```

　最後に StackStorm の各サービスを再起動させれば、冗長環境の構築は完了です。  

```sh
vagrant@st2-secondary:~$ sudo st2ctl restart
```

　ここまでで StackStorm は冗長な構成となり、例え `st2-secondary` ノードが停止しても、StackStorm 全体に影響を及ぼすことはありません。  
　尚、参考までに DMM.com ラボの環境では更にプライマリノードから RabbitMQ、MongoDB を外に出し、PostgreSQL を MySQL に変えた以下の構成を採っています。  

![DMM.com ラボの StackStorm 環境](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/dmm-architecture.png)

　このように StackStorm ノードから状態を保つ要素を外に出しステートレス (Stateless) にすることで、StackStorm ノードをスケールさせることができるようになります。  

## High Available な環境構築
　これまで StackStorm の使い方や機能拡張の方法など、どちらかというと開発寄りな内容について解説してきましたが、ここからは StackStorm の運用のポイントについて解説します。  
　本稿の冒頭で StackStorm が Scalable で High Available なアーキテクチャであることを説明しました。ここでは、冗長構成な StackStorm における運用の注意点について簡単に解説します。  

### 冗長構成な StackStorm の構築

　ここで冗長構成な StackStorm を構築します。  
　[StackStorm のドキュメント](https://docs.stackstorm.com/reference/ha.html#reference-ha-setup) に、以下の構成での構築方法が詳細に解説されていますが、ここでは簡単のために別の方法を紹介します。  

![StackStorm HA reference deployment](https://docs.stackstorm.com/_images/st2-deployment-multi-node.png)
(出典：[High availability deployment | StackStorm 2.0.1 documentation](https://docs.stackstorm.com/reference/ha.html))

　ここでは、最初に構築したノードをプライマリノードとし、これから構築するノードをセカンダリノードとする環境を構築します。図にすると以下のようになります。  

![StackStorm 冗長構成](p7)

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

![DMM.com ラボの StackStorm 環境](P11)

　このように StackStorm ノードから状態を保つ要素を外に出しステートレス (Stateless) にすることで、StackStorm ノードをスケールさせることができるようになります。  

## 冗長構成な StackStorm の運用の注意点

　さて StackStorm ノードが冗長化し、更にステートレスになるとスケールできて嬉しいのですが、何も考えずに StackStorm ノードを増やしてゆくと、トリガが重複して引かれるという問題が発生します。これが一体どうよう問題で、どのように回避できるかについて最後に解説します。  

　まず、トリガが重複して引かれるとはどういうことかについて見て行きます。冗長化された環境で、先ほど作成したセンサ `mypack.DirectorySensor` にファイル作成のイベントを通知させます。st2-node もしくは st2-secondary のどちらかで、以下の操作を行ってください。  

```bash
$ sudo touch /opt/stackstorm/packs/hoge
```

　すると、それぞれのノードでアクション `mypack.output_context` が実行され、両ノードの `/tmp/output` に以下のような結果が出力されることが確認できると思います（もしかすると、どちらか一方のノードで２回出力されるかもしれません。その理由は本編の最後で説明します）。  

```
[1480385994.27] (created) /opt/stackstorm/packs/hoge
[1480386001.36] (created) /opt/stackstorm/packs/mypack/actions/output_context.pyc
```

　これは、ファイル `hoge` を作成したイベントを全てのノードで検知し、それぞれのノードでトリガ `mypack.changed_file` が引かれ、ルール `mypack.mypack_test` に従ってアクションを実行した結果起こりました。以下はこの状況を表した図になります。  

![StackStorm の裏側の出来事](p11)

　ここでの問題は、イベントの発生源が１つにも関わらずアクションが２回実行されたことです。例ではログにイベントの結果を吐き出すだけなので大したことはありませんが、イベントに対してアラートを発報したり、Public Cloud のインスタンスを起動するような仕組みの場合には深刻です。またノードが増える毎に深刻の度合いが増して行きます。  
　この問題に対して StackStorm は [Partitioning Sensors](https://docs.stackstorm.com/reference/sensor_partitioning.html) という仕組みを提供しています。これは、ノード毎に稼働するセンサを管理する手法で、図で表すと先ほどの状況を以下のようにすることができます。  

![Partitioning Sensors を用いた場合](p12)

　このようにノード毎にセンサを分離させるとで、複数のノードがある環境下で発生したイベントに対して、センサ、トリガ、アクションを全て一意に対応させることが出来るようになります。  

### Partitioning Sensors の設定
　ここでは、先の図で示したように `st2-node` で AWS のイベントを監視するセンサを動かし、`st2-secondary` で `mypack.DirectorySensor` を動かすように設定します。  

　まず各ノードの StackStorm 設定ファイル `/etc/st2/st2.conf` を以下の通り編集します。  

```diff
--- etc/st2/st2.conf	2016-11-29 03:41:03.660109005 +0000
+++ /etc/st2/st2.conf	2016-11-29 03:41:25.416109660 +0000
@@ -84,3 +84,7 @@
 
 [database]
 host = 192.168.0.100
+
+[sensorcontainer]
+sensor_node_name = st2-node
+partition_provider = name:file, partition_file:/usr/local/etc/partition_sensor.conf
```
[st2-node の編集結果]

```diff
--- etc/st2/st2.conf	2016-11-29 03:37:29.488251541 +0000
+++ /etc/st2/st2.conf	2016-11-29 03:40:49.124260627 +0000
@@ -84,3 +84,7 @@
 
 [database]
 host = 192.168.0.100
+
+[sensorcontainer]
+sensor_node_name = st2-secondary
+partition_provider = name:file, partition_file:/usr/local/etc/partition_sensor.conf
```
[st2-secondary の編集結果]

　`sensor_node_name` はノードの識別子で、ノードを一意に特定できれば任意の文字列を指定できます。ここでは、各ノードのホスト名を指定しています。  
　`parittion_provider` では、センサを分離する方法を規定します。ここでは、後述するファイル `/opt/stackstorm/packs/partition_sensor.conf` にどのノードでどのセンサを動作させるかを記載する設定を行っています。この他にも、設定ファイルに直接動作させるセンサを記述する方法などがあります。  

　続いて、ノードとセンサの対応を記載した以下の設定を記述したファイルを、両ノードの `/usr/local/etc/partition_sensor.conf` に設定します。  

```yaml
---
st2-node:
    - aws.ServiceNotificationsSensor
    - aws.AWSSQSSensor

st2-secondary:
    - mypack.DirectorySensor 
```

　最後に、両ノードの StackStorm を再起動をさせれば設定は完了です。  

```sh
$ sudo st2ctl restart
```

### 動作確認
　それでは、再度センサ `mypack.DirectorySensor` にイベントを通知させます。どちらかのノードで、以下のように先ほど作成したファイルを削除してみてください。  

```bash
$ sudo rm /opt/stackstorm/packs/hoge
```
　すると、`st2-node` あるいは `st2-secondary` のどちらかのファイル `/tmp/output` に以下の結果が出力されるはずです。  
```
[1480385994.27] (deleted) /opt/stackstorm/packs/hoge
```
　更にここで注意が必要なのは、センサがファイル `/opt/stackstorm/packs/hoge` の削除を検知してトリガ `mypack.changed_file` を引く処理を行うノードは `st2-secondary` ですが、アクションを実行するノードは不定（`st2-node` か `st2-secondary` かわからない）ということです。  

　その理由は、センサは分離されましたが、ワーカは全てのノードで動作しているためです。  
　入門編の [StackStorm の特徴]() で示した StackStorm の内部アーキテクチャの図を思い出してください。センサとワーカが別々に動作しており、それらが MQ を介して接続されています。[StackStorm の機能拡張]() で解説した通り、センサは `SensorService` の `dispatch` メソッドを実行してトリガを引きます。こうして送られた通知は `st2rulesengine` というプロセスに送られ、トリガに紐付けられたルールが無いかを確認します。そして当該トリガに紐付くルールを見つけると、アクションの実行命令をワーカ (`st2actionrunner` プロセス) に MQ を介して送ります。アクション実行命令を受け取ったワーカは、[runner_type](https://docs.stackstorm.com/actions.html#available-runners) に従った形式でアクションを実行します。  
　こうした仕組みによって、アクションの実行をワーカノードで分散できると共に、ワーカノードを動的にスケールさせることもできます。  

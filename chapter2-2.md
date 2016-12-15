## High Available な環境構築
　これまで StackStorm の使い方や機能拡張の方法など、開発寄りな内容について解説してきましたが、ここからは StackStorm の運用のポイントについて解説します。  
　本稿の冒頭で StackStorm が Scalable で High Available なアーキテクチャであることを説明しました。ここでは、冗長構成な StackStorm における運用に焦点を移して解説します。  

### 冗長構成な StackStorm の構築
　まずは冗長構成な StackStorm を構築します。  
　[StackStorm のドキュメント](https://docs.stackstorm.com/reference/ha.html#reference-ha-setup) に、以下の構成での構築方法が解説されていますが、ここでは簡単のために別の方法を紹介します。  

![StackStorm HA reference deployment](https://docs.stackstorm.com/_images/st2-deployment-multi-node.png)
(出典：[High availability deployment | StackStorm 2.0.1 documentation](https://docs.stackstorm.com/reference/ha.html))

　ここでは、最初に構築したノード (プライマリノード) に加えて、もう一台のノード (セカンダリノード) を追加することで、冗長な StackStorm 環境を構築します。最終的に、以下のような環境を構築します (なお以降では Mistral を利用しないため PostgreSQL についての設定は省略します) 。 

![StackStorm 冗長構成](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/ha-architecture.png)

　上記の構成で StackStorm の各サービスは冗長化しますが「プライマリノードが単一障害点になるじゃないか！」と思う鋭い読者もいるかもしれません。  
　確かに現状はその通りですが、[MongoDB のレプリケーション](https://docs.mongodb.com/manual/replication/) や [RabbitMQ のクラスタ設定](https://www.rabbitmq.com/clustering.html) を利用することで、完全に単一障害点を無くす構成の StackStorm 環境を構築することもできます (本章の最後で単一障害点を無くした DMM.com ラボにおける構成を紹介します)。ただ、これらの内容については本稿の趣旨を超えるためここでは割愛します。ただし RabbitMQ のクラスタ設定については、StackStorm における RabbitMQ のクラスタ指定の説明の都合上、簡単に解説します。 

### プライマリノードの設定変更
　冗長構成の環境を構築する上で、プライマリノードに対して、セカンダリノードから MongoDB へアクセスを許可する設定、及び RabbitMQ のクラスタ設定、更に NFS の設定を行います。それぞれの設定方法について順に解説します。  

#### MongoDB の設定
　まず MongoDB にセカンダリノードからアクセスできるよう設定ファイル (/etc/mongodb.conf) ファイルを以下の通り修正し、サービスを再起動させます。  

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
```

#### RabbitMQ の設定
　続いて RabbitMQ のクラスタ環境を構築するために、ノード間で共通の [Erlang Cookie] を設定します。Erlang Cookie は RabbitMQ の実装言語 Erlang の言語機能で、複数ノード間での相互接続を許可するためのセキュリティ機構です。Cookie を指定した場合、Erlang インタプリタは同一の設定値を持つノード同士でのみ接続することができます。  
　ここでは、以下の通り RabbitMQ の Erlang Cookie を再設定し、RabbitMQ を再起動させます。  

```
vagrant@st2-node:~$ sudo service rabbitmq-server stop
vagrant@st2-node:~$ sudo bash -c 'echo stackstorm-mq-cluster > /var/lib/rabbitmq/.erlang.cookie'
vagrant@st2-node:~$ sudo service rabbitmq-server start
```

　また、お互いのノードでホスト名の名前解決ができるよう `/etc/hosts` に以下の２行を追加します。２行目はこのあと構築するセカンダリノードの設定値になります。  

```
vagrant@st2-node:~$ sudo bash -c 'cat <<EOS >> /etc/hosts
192.168.0.100     st2-node
192.168.0.101     st2-secondary
EOS
'
```

　クラスタ設定の方法については、セカンダリノードを構築する際に解説します。  

#### NFS の設定
　StackStorm で冗長構成を組む場合、StackStorm が参照するコンテンツディレクトリ (`/opt/stackstorm/` ディレクトリ配下の `packs` と `virtualenvs` の２つ) をノード間で共有させる必要があります。ここではプライマリノードに NFS サーバを構築し、セカンダリノードからコンテンツディレクトリがマウントできるようにします。  

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

　StackStorm をオールインワンインストールをした場合、コンテンツディレクトリの所有者（ユーザ・グループ）は root/st2packs に設定されます。マウント先でも同様に扱えるようにするため `anonuid` と `anongid` オプションに、それぞれ root の uid と st2packs の gid を設定します。なお st2packs の gid は以下の方法で確認できます。  

```sh
vagrant@st2-node:~$ grep 'st2packs' /etc/group
st2packs:x:999:st2
vagrant@st2-node:~$ 
```

　最後に NFS サーバを再起動させれば完了です。  

```
vagrant@st2-node:~$ sudo service nfs-kernel-server restart
```

### セカンダリノードの構築
　続いて、セカンダリノードの構築を行います。  
　基本編の [StackStorm のインストール](https://github.com/userlocalhost/st2-draft/blob/master/chapter1-3.md) で解説した方法で Vagrant からセカンダリノードを作成し、StackStorm をインストールします。その際 Vagrantfile に設定するプライベートネットワークの IP アドレスを、プライマリノードに設定したものと違う値を設定してください。ここでは以下の通り Vagrantfile を修正して、セカンダリノードを作成します。  

```diff
$ diff Vagrantfile.orig Vagrantfile
--- Vagrantfile.orig    2016-11-16 19:06:55.000000000 +0900
+++ Vagrantfile 2016-12-05 16:49:56.000000000 +0900
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
$ vagrant up
$ vagrant ssh
```

　セカンダリノードが立ち上がったら、作成したノードで StackStorm をインストールします。  

```
vagrant@st2-secondary:~$ curl -sSL https://stackstorm.com/packages/install.sh | bash -s -- --user=st2admin --password=password
```

#### RabbitMQ クラスタ設定
　プライマリノードで実行した時と同じように、セカンダリノードでも `/etc/hosts` の修正、及び RabbitMQ の Erlang Cookie の再設定を行います。  

```
vagrant@st2-secondary:~$ sudo bash -c 'cat <<EOS >> /etc/hosts
192.168.0.100     st2-node
192.168.0.101     st2-secondary
EOS
'
```

```
vagrant@st2-secondary:~$ sudo service rabbitmq-server stop
vagrant@st2-secondary:~$ sudo bash -c 'echo stackstorm-mq-cluster > /var/lib/rabbitmq/.erlang.cookie'
vagrant@st2-secondary:~$ sudo service rabbitmq-server start
```

　２つのノードで同じ Cookie の値を持つ RabbitMQ ノードを起動させたところで、クラスタ設定を行います。クラスタ設定を行うことで、クラスタのノード間で、メッセージキューを除く全ての管理・設定情報が共有されるようになります。クラスタ設定はどちらか一方のノードでのみ行えば良いです。ここではセカンダリノードで以下のコマンドを実行し、プライマリノードとクラスタを組みます。  

```
vagrant@st2-secondary:~$ sudo rabbitmqctl stop_app
vagrant@st2-secondary:~$ sudo rabbitmqctl join_cluster rabbit@st2-node
vagrant@st2-secondary:~$ sudo rabbitmqctl start_app
```

　続いて RabbitMQ に外部からのアクセス用のユーザを追加します。これまでは、デフォルトで作成される `guest` アカウントでログインしていましたが、[`guest` アカウントはローカルホストからしかアクセス出来ない](https://www.rabbitmq.com/access-control.html) ため、ここで別アカウント `st2` を作成し、アクセス権限の付与を行います。  
　既にクラスタ設定を行っているので、どちらか一方のノードで作成・設定作業をすれば、もう一方のノードにも同じ設定が反映されます。  

```
vagrant@st2-secondary:~$ sudo rabbitmqctl add_user st2 passwd
vagrant@st2-secondary:~$ sudo rabbitmqctl set_permissions st2 ".*" ".*" ".*"
```

#### NFS の設定
　続いてプライマリノードのコンテンツディレクトリを NFS でマウントし、実行環境及び静的データをプライマリノードと共有します。`/etc/fstab` を以下の通り修正し、それぞれのマウントポイントで指定したディレクトリをマウントをします。  

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

### StackStorm 設定ファイルの修正
　最後に、これらの設定変更に伴う StackStorm の設定ファイルの修正を行います。**プライマリ、セカンダリの両ノード**で以下の通り StackStorm の設定ファイル `/etc/st2/st2.conf` を修正します。  

```diff
--- etc/st2/st2.conf.orig       2016-12-07 07:31:31.606370802 +0000
+++ /etc/st2/st2.conf   2016-12-07 07:48:08.470374159 +0000
@@ -73,7 +73,7 @@
 ssh_key_file = /home/stanley/.ssh/stanley_rsa
 
 [messaging]
-url = amqp://guest:guest@127.0.0.1:5672/
+cluster_urls = amqp://st2:passwd@192.168.0.100:5672/, amqp://st2:passwd@192.168.0.101:5672/
 
 [ssh_runner]
 remote_dir = /tmp
@@ -81,3 +81,6 @@
 
 [keyvalue]
 encryption_key_path = /etc/st2/keys/datastore_key.json
+
+[database]
+host = 192.168.0.100
```

　RabbitMQ の接続先を設定していた `url` パラメータに代わり `cluster_urls` で、クラスタ設定を行った RabbitMQ ノード群の設定を行います。また `[database]` セクションの `host` パラメータでは、参照する MongoDB の接続先を指定しています。  
　上記の設定を行ったら、両ノードで以下の通り StackStorm を再起動させます。  

```
$ sudo st2ctl restart
```

### 動作確認
　ここまでの設定で冗長構成な StackStorm 環境を構築することができました。以下のように、一方のノードでインストールした Pack がもう一方のノードにも反映されることが確認できます。  

```
vagrant@st2-node:~$ st2 pack install github
```

```
vagrant@st2-secondary:~$ st2 pack list | grep github
| github  | st2 content pack containing github integrations | 0.6.0   | StackStorm, Inc. |
vagrant@st2-secondary:~$
```

　上記のように、セカンダリノードで github pack の情報が得られれば成功です。またセカンダリノードを停止させても、Pack のインストールから、センサのイベント監視、アクションの実行を継続的に行うことができます。  
　
### DMM.com ラボにおける StackStorm 環境
　参考として、DMM.com ラボにおける StackStorm 環境を以下で紹介します。  

![DMM.com ラボの StackStorm 環境](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/dmm-architecture.png)

　DMM.com ラボの環境では、プライマリノードから更に RabbitMQ、MongoDB を外に出し、PostgreSQL を MySQL に変えた構成を採っています。また RabbitMQ は、上述のクラスタ設定に加えて、メッセージキューのデータもノード間で複製する [HA Queue](https://www.rabbitmq.com/ha.html) の設定を行っています。これによって、当該環境の可用性をより高めることができます。  
　更に、コンテンツディレクトリも可用性の高い共有ストレージに置くことで、StackStorm ノードから状態を持つ要素を完全に無くすことができるようになります。  
　このように StackStorm ノードをステートレス (stateless) にすることで、StackStorm ノードをスケールさせることができるようになり、負荷の増加に対しても柔軟に対応できるようになります。  

## StackStorm のインストール
　公式ドキュメントの [Installation](https://docs.stackstorm.com/install/index.html) に沿って、一般にオールインワン (All-in-one) と呼ばれる StackStorm の全サービス、及び関連する全てのミドルウェアを一つのノードにインストールする方法を紹介します。  
　ノードは vagrant で用意します。vagrant 自体のインストール方法については、[過去の記事](http://codezine.jp/article/detail/8255?p=2) で詳しく解説されておりますので、そちらをご参照ください。  

　まずは以下のコマンドで Vagrantfile を作成します。ここでは公式ドキュメントの [System Requirements](https://docs.stackstorm.com/install/system_requirements.html) に沿って Ubuntu14.04 の VM を作成します。  
```
$ vagrant init ubuntu/trusty64
```
　ノードのメモリサイズが小さいとリソース不足でビルド処理が失敗することがあるため、以下のとおり Vagrantfile を編集し、仮想 CPU コア数と割り当てるメモリサイズを変更します。また、後述する冗長構成な StackStorm 環境の構築のため、仮想 NIC の設定追加も併せて行います。  
```diff
--- Vagrantfile.orig	2016-11-16 19:06:55.000000000 +0900
+++ Vagrantfile	2016-11-28 14:46:04.000000000 +0900
@@ -27,6 +27,8 @@
   # Create a private network, which allows host-only access to the machine
   # using a specific IP.
   # config.vm.network "private_network", ip: "192.168.33.10"
+  config.vm.hostname = "st2-node"
+  config.vm.network "private_network", ip: "192.168.0.100"
 
   # Create a public network, which generally matched to bridged network.
   # Bridged networks make the machine appear as another physical device on
@@ -50,6 +52,10 @@
   #   # Customize the amount of memory on the VM:
   #   vb.memory = "1024"
   # end
+  config.vm.provider "virtualbox" do |vb|
+    vb.cpus = 2
+    vb.memory = "2048"
+  end
   #
   # View the documentation for the provider you are using for more
   # information on available options.
```
　編集後、以下のコマンドを実行し VM を構築します。  
```
$ vagrant up
```
　続いて以下のコマンドで VM にログインします。また以降の操作は、ログインした VM 内で実行します。  
```
$ vagrant ssh
```
　最後に StackStorm のインストールコマンドを実行します。引数で渡している `--user` と `--password` は、初期アカウントのユーザ名とパスワードになり、任意の値を設定できます。  
```
$ curl -sSL https://stackstorm.com/packages/install.sh | bash -s -- --user=st2admin --password=password
```
　以下の出力が得られれば、インストールは成功です。  
```
███████╗████████╗██████╗      ██████╗ ██╗  ██╗
██╔════╝╚══██╔══╝╚════██╗    ██╔═══██╗██║ ██╔╝
███████╗   ██║    █████╔╝    ██║   ██║█████╔╝ 
╚════██║   ██║   ██╔═══╝     ██║   ██║██╔═██╗ 
███████║   ██║   ███████╗    ╚██████╔╝██║  ██╗
╚══════╝   ╚═╝   ╚══════╝     ╚═════╝ ╚═╝  ╚═╝

  st2 is installed and ready to use.

Head to https://YOUR_HOST_IP/ to access the WebUI

Don't forget to dive into our documentation! Here are some resources
for you:

* Documentation  - https://docs.stackstorm.com
* Knowledge Base - https://stackstorm.reamaze.com

Thanks for installing StackStorm! Come visit us in our Slack Channel
and tell us how it's going. We'd love to hear from you!
http://stackstorm.com/community-signup
vagrant@vagrant-ubuntu-trusty-64:~$ 
```

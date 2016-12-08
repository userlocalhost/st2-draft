## 冗長構成な StackStorm の運用の注意点

　さて StackStorm ノードが冗長化し、更にステートレスになるとスケールできて嬉しいのですが、何も考えずに StackStorm ノードを増やしてゆくと、運用上困った事態が起こります。  
　どういう事が起こるかを知るために、前節で構築した冗長環境で、[StackStorm の機能拡張](https://github.com/userlocalhost/st2-draft/blob/master/chapter2-1.md) で作成した `mypack.DirectorySensor` にファイル作成のイベントを検知させます。  
　st2-node もしくは st2-secondary のどちらかで、以下の操作を行ってください。  

```bash
$ sudo touch /opt/stackstorm/packs/hoge
```

　すると、それぞれのノードでアクション `mypack.output_context` が実行され、両ノードの `/tmp/output` に以下のような結果が出力されることが確認できると思います（もしかすると、どちらか一方のノードで２回出力されるかもしれません。その理由は本編の最後で説明します）。  

```
[1480385994.27] (created) /opt/stackstorm/packs/hoge
[1480386001.36] (created) /opt/stackstorm/packs/mypack/actions/output_context.pyc
```

　これは、ファイル `hoge` を作成したイベントを全てのノードで検知し、それぞれのノードでトリガ `mypack.changed_file` が引かれ、ルール `mypack.mypack_test` に従ってアクションを実行した為に、このようになりました。以下はこの状況を表した図になります。  

![StackStorm の裏側の出来事](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/ha-problem.png)

　ここでの問題は、イベントの発生源が１つにも関わらずアクションが２回実行されたことです。  
　例ではログにイベントの結果を出力するだけなので大した影響はありませんが、イベントに対してアラートを発報したり、Public Cloud のインスタンスを起動するような仕組みの場合には深刻です。またノードが増える毎にその度合いは増して行きます。  
　StackStorm はこの問題を回避する手段として [Partitioning Sensors](https://docs.stackstorm.com/reference/sensor_partitioning.html) という仕組みを提供しています。各ノードで Partitioning Sensors を設定することで、ノード毎に稼働するセンサを管理・選択できるようになります。これによって、先ほどの状況を以下のようにすることができます。  

![Partitioning Sensors を用いた場合](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/partitioning-sensors.png)

　このようにノード毎にセンサを分離させるとで、複数の StackStorm ノードがある環境下で発生したイベントに対して、センサ、トリガ、アクションを一意に紐付けることが出来るようになります。  

### Partitioning Sensors の設定
　ここで Partitioning Sensors を設定し、先の図で示したように `st2-node` で AWS のイベントを監視するセンサを動かし、`st2-secondary` でセンサ `mypack.DirectorySensor` を動かすように設定します。  

　まず各ノードの StackStorm 設定ファイル `/etc/st2/st2.conf` を以下の通り編集します。  

```diff
--- etc/st2/st2.conf.orig	2016-12-07 11:58:32.021642436 +0000
+++ /etc/st2/st2.conf	2016-12-07 11:59:51.177201387 +0000
@@ -14,6 +14,8 @@
 
 [sensorcontainer]
 logging = /etc/st2/logging.sensorcontainer.conf
+sensor_node_name = st2-node
+partition_provider = name:kvstore
 
 [rulesengine]
 logging = /etc/st2/logging.rulesengine.conf
```
[st2-node の編集結果]

```diff
--- etc/st2/st2.conf.orig	2016-12-07 11:58:46.379088514 +0000
+++ /etc/st2/st2.conf	2016-12-07 11:59:22.409093433 +0000
@@ -14,6 +14,8 @@
 
 [sensorcontainer]
 logging = /etc/st2/logging.sensorcontainer.conf
+sensor_node_name = st2-secondary
+partition_provider = name:kvstore
 
 [rulesengine]
 logging = /etc/st2/logging.rulesengine.conf
```
[st2-secondary の編集結果]

　Partitioning Sensors の設定は `[sensorcontainer]` セクション以下で行います。  
　`sensor_node_name` は StackStorm ノードの識別子で、ノードを一意に特定できれば任意の文字列を指定できます。ここでは、各ノードのホスト名を指定します。  
　`parittion_provider` では、当該ノードで動かすセンサを '設定する方法' を設定します。ここでは、MongoDB で管理されるデータストアに、どのノードでどのセンサを動作させるかの設定を記述します。この他にも、[ファイルで管理する方法](https://docs.stackstorm.com/reference/sensor_partitioning.html#file) などもあります。  

　設定ファイルで Partitioning Sensors の設定方法を決めたところで、具体的にどのノードでどのセンサを動作させるかの設定を行います。先に示した図の通り、 st2-node で aws pack のセンサ (`aws.ServiceNotificationsSensor` と `aws.AWSSQSSensor`) を起動させ、st2-secondary で mypack のセンサ (`mypack.DirectorySensor`) を起動させるよう、以下のコマンドでデータストアに設定します。  

```
$ st2 key set st2-node.sensor_partition "aws.ServiceNotificationsSensor, aws.AWSSQSSensor"
$ st2 key set st2-secondary.sensor_partition "mypack.DirectorySensor"
```

　データストアに設定した内容は、以下のコマンドで確認できます。なお、データストアはどちらも st2-node の MongoDB を参照しているため、どちらのノードで実行しても同じ結果が得られます。  

```
$ st2 key list
+--------------------------------+--------------------------------------------------+--------+-----------+--------------+------+------------------+
| name                           | value                                            | secret | encrypted | scope        | user | expire_timestamp |
+--------------------------------+--------------------------------------------------+--------+-----------+--------------+------+------------------+
| st2-secondary.sensor_partition | mypack.DirectorySensor                           | False  | False     | st2kv.system |      |                  |
| st2-node.sensor_partition      | aws.ServiceNotificationsSensor, aws.AWSSQSSensor | False  | False     | st2kv.system |      |                  |
+--------------------------------+--------------------------------------------------+--------+-----------+--------------+------+------------------+
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
　すると、`st2-node` あるいは `st2-secondary` のどちらかのファイル `/tmp/output` で、以下のような結果が出力されるはずです。  
```
[1480385994.27] (deleted) /opt/stackstorm/packs/hoge
```
　更にここで注意が必要なのは、センサがファイル `/opt/stackstorm/packs/hoge` の削除を検知してトリガ `mypack.changed_file` を引く処理を行うノードは Partitioning Sensors で設定した `st2-secondary` ですが、アクションを実行するノードは不定（`st2-node` か `st2-secondary` かわからない）ということです。  

　その理由は、ワーカプロセス (st2actionrunner) が全てのノードで動作しているためです。StackStorm では以下の図のように、センサプロセスとワーカプロセスが別々に動作しており、それらが MQ を介して接続されています。  

![イベント検知からアクション実行までの各プロセスの処理の流れ](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/processing-flow.png)

　[StackStorm の機能拡張](https://github.com/userlocalhost/st2-draft/blob/master/chapter2-1.md) で解説した通り、センサは `SensorService` の `dispatch` メソッドを実行してトリガを引きます。こうして送られた通知は `st2rulesengine` というプロセスに送られ、トリガに紐付けられたルールが無いかを確認します。そして当該トリガに紐付くルールを見つけると、アクションの実行命令 (ActionExecution) をワーカプロセス `st2actionrunner` に MQ を介して送ります。アクション実行命令を受け取ったワーカは、[runner_type](https://docs.stackstorm.com/actions.html#available-runners) に従った形式でアクションを実行します。  

　このため、アクションが実行されるノードは一意に決まりませんが、逆にこの仕組みによって、アクションを複数のワーカノードで分散できると共に、動的にスケールさせることが出来るようになります。  

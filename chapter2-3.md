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

![StackStorm の裏側の出来事](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/ha-problem.png)

　ここでの問題は、イベントの発生源が１つにも関わらずアクションが２回実行されたことです。例ではログにイベントの結果を吐き出すだけなので大したことはありませんが、イベントに対してアラートを発報したり、Public Cloud のインスタンスを起動するような仕組みの場合には深刻です。またノードが増える毎に深刻の度合いが増して行きます。  
　この問題に対して StackStorm は [Partitioning Sensors](https://docs.stackstorm.com/reference/sensor_partitioning.html) という仕組みを提供しています。これは、ノード毎に稼働するセンサを管理する手法で、図で表すと先ほどの状況を以下のようにすることができます。  

![Partitioning Sensors を用いた場合](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/partitioning-sensors.png)

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

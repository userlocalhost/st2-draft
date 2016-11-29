## StackStorm の機能拡張
　ここでは、これまでに解説してきたセンサ、トリガ、そしてアクションを自作する方法を解説します。  
　ただ、おおよその外部アプリケーションと連携するために必要となるパック群は [StackStorm Community Repository](https://github.com/StackStorm/st2contrib) で公開されており、pack をユーザ自身が作らなければならない事態はあまりないかもしれませんが、ここでの内容によって、従来の自動化ツールを StackStorm に統合することができれば、それらの資産を活かすことができると思います。  

　以下では例として、ローカルマシンのファイルシステムのディレクトリの変更を監視するセンサ `DirectorySensor` とファイル更新が行われた際に引かれるトリガ `changed_file`、及び入力されたパラメータをログに吐き出すアクション `output_context` を実装します。  
　あまり面白みの無い仕組みですが、これらをどのように実装するのかを理解するにはちょうどよいかと思います。  

### pack を作る
　まずは [st2sdk](https://github.com/StackStorm/st2sdk) を使って pack の雛形を作成します。以下の手順で `st2sdk` をインストールし、オリジナルの pack (`mypack`) を作成します。  

```
$ sudo apt-get install python-pip   ## pip コマンドをインストール
$ sudo pip install st2sdk           ## st2sdk をインストール
$ st2sdk bootstrap mypack           ## pack の雛形を作成
```

　`st2sdk` のサブコマンド `bootstrap` によって、パックの定義ファイル `pack.conf` や設定ファイル `config.yaml` など、最低限必要なファイルが自動生成されます。  
　尚、以降で解説するコードを含む `mypack` 全体を以下のリポジトリで公開しています。  

* [https://github.com/userlocalhost2000/st2-pack-example](https://github.com/userlocalhost2000/st2-pack-example)

### センサとトリガを作る
　続いてセンサとトリガを作成します。コードを見る前にセンサに関するソフトウェアの構造について把握したいと思います。以下の図はセンサのソフトウェアアーキテクチャを表しています。  

![センサのソフトウェアアーキテクチャ](p8)

　Sensor は外部システムのイベントを検知するユーザ定義の処理で、この後でその実装方法を示します。SensorService は Sensor がトリガやデータストア、設定ファイルなどにアクセスするための仕組みになります。Sensor は SensorService の `get_value` / `set_value` メソッド等を通じて MongoDB で管理されるデータストアのデータを扱うことや `dispatch` メソッドを通じて指定したトリガを引くことができます。  
　また StackStorm は各センサごとにプロセスを立ち上げます。従って、ユーザ定義の処理がクラッシュした場合でも、他の StackStorm サービスに対して影響を及ぼすことはありません。  

　センサもワークフロー同様、メタデータと実装から成り立っています。以下は冒頭のセンサのメタデータファイルの全文になります。  

```yaml
---
class_name: "DirectorySensor"
entry_point: "directory_sensor.py"
description: "An example sensor to know how to implement it"
poll_interval: 30
trigger_types:
    - name: "changed_file"
      description: "Trigger which indicates a change of trigger"
      payload_schema:
          type: "object"
          properties:
              paths:
                  type: "string"
              time:
                  type: "string"
              status:
                  type: "string"
```

　`trigger_types` 以下で、このセンサに紐付くトリガ `changed_file` とトリガのパラメータを規定しています。センサはトリガを引く際にここで規定したパラメータを渡す処理を行います。こうすることで、ユーザがルールを記述する際にスムーズにパラメータ変換を行うことができます。  
　`entry_point` はセンサの実装ファイルになります。このファイルはパックのセンサディレクトリ `/opt/stacksotm/packs/<パック名(この場合 'mypack')>/sensors` からの相対パスを指定します。また `class_name` で後述するセンサを実装したクラスを指定します。  

　それでは、センサの実装を見て行きます。以下が実装ファイル `directory_sensor.py` の抜粋になります(全文は、先述のソースコードをご参照ください)。  

```python
from st2reactor.sensor.base import PollingSensor


class DirectorySensor(PollingSensor):
    def setup(self):
        self._logger = self._sensor_service.get_logger(__name__)

        # initialize directory info
        self._cached_dirinfo = self._get_dirinfo()

    def poll(self):
        current_dirinfo = self._get_dirinfo()

        # notify deleted files
        for path in (set(self._cached_dirinfo) - set(current_dirinfo)):
            self._dispatch_trigger(path, self._cached_dirinfo[path], 'deleted')

        # notify created files
        for path in (set(current_dirinfo) - set(self._cached_dirinfo)):
            self._dispatch_trigger(path, current_dirinfo[path], 'created')

        # notify modified files
        for path in [x for x in (set(self._cached_dirinfo) & set(current_dirinfo))
                             if self._cached_dirinfo[x] != current_dirinfo[x]]:
            self._dispatch_trigger(path, current_dirinfo[path], 'modified')

        # update cache data
        self._cached_dirinfo = current_dirinfo

...(略)...

    def _get_dirinfo(self):
        dirinfo = {}
        for target in self._get_sensor_config('directories'):
            self._do_get_dirinfo(target, dirinfo)

        return dirinfo

...(略)...

    def _dispatch_trigger(self, path, time, status):
        payload = {
            'path': path,
            'time': time,
            'status': status,
        }
        self._sensor_service.dispatch(trigger="mypack.changed_file", payload)
```

　センサは、何らかの仕組みによって外部システムのイベントを受動的に受け取るパッシブセンサ `Sensor` か、センサ自体が外部システムを参照して能動的にイベントを取得しに行くアクティブセンサ `PollingSensor` のいづれかに分類されます。`DirectorySensor` は定期的にファイルシステムを確認して差分を確認するアクティブセンサを実装するため `PollingSensor` を継承します。逆に [inotify](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/inotify.7.html) などの仕組みによってファイルシステムの変更を監視するセンサを実装する場合には、パッシブセンサの親クラス `Sensor` を継承するのが良いと思います。  

　センサは、プロセスが実行された時や初期化された時など、幾つかのコールバックを受け取ることができます。ここでは、センサが初期化された後に１度だけ呼ばれるコールバックメソッド `setup` と、メタデータファイルの `poll_interval` パラメータで指定した間隔 (単位は秒) 毎に呼び出される `poll` メソッドを実装しています。  
　処理の中身は、まず初期化処理においてプライベートメソッド `_get_dirinfo` を呼び出し、当該パックの設定ファイルで指定したディレクトの各ファイルの更新時間を取得する処理 `_do_get_dirinfo` を実行し、結果を返します (`_do_get_dirinfo` のコードの説明は割愛します)。以前に解説したパックの設定ファイルの配置ルール (`/opt/stackstorm/packs/<パック名>/config.yaml`) を思い出してください。以下に、このパックの設定ファイルの抜粋を示します。  

```yaml
---
sensor:
    directories:
        - /opt/stackstorm
```

　続いて定期的に呼び出される `poll` において、同じように `_get_dirinfo` を呼び出し、前回実行時との差分 (作成、削除、更新されたファイル) を取得し、差分があった場合に `_dispatch_trigger` を呼び出し、`mypack.changed_file` トリガを引きます。その際、メタデータファイルで規定したフォーマットのハッシュオブジェクトにデータをまとめてから、引数で渡します。  
　こうして通知されたイベントが MQ を経由し、当該トリガに関連付けられたルールが存在する場合には、それに紐付くアクションがワーカノードで実行されます。  

### Action を作る
　最後にアクション `output_context` を作成します。アクションもメタデータファイルと定義ファイルの２つから構成されます。以下にメタデータファイルの全文を記載します。  

```yaml
---
name: output_context
pack: mypack
runner_type: run-python
entry_point: output_context.py
description: 'An example action to know how it works'
enabled: true
parameters:
  context:
    type: string
    default: ''
```

　メタデータファイルの構成は、入門編の最後で解説したワークフローとほとんど同じです。大きく違う点は、アクションを Python スクリプトで記述するために `runner_type` に `run-python` を指定しているくらいで、他は、ワークフローのケースと同様に `entry_point` でアクション定義 (Python スクリプト) ファイルを指定して、`parameters` でアクションに渡すパラメータを指定しています。  
　続いてアクション定義ファイルの全文を以下に記載します。  

```python
import os

from st2actions.runners.pythonrunner import Action


class OutputContext(Action):
    def run(self, context):
        output_path = self.config.get('log', None)

        if output_path:
            try:
                with open(output_path, 'a') as file:
                    file.write(context + '\n')
            except IOError as err:
                return (False, "IOError is occurred (%s)" % (err))

            return (True, "This processing is succeeded.")
        else:
            return (False, "The output filepath is invalid.")
```

　ユーザ定義アクション `OutputContext` は、センサ同様に StackStorm が用意したベースクラス `Action` を継承します。内部では、パックの設定やデータストア、ログにアクセスするための仕組みを提供する `ActionService` オブジェクトへの参照を持っており、これらと連携したアクションを簡単に記述することができます。  
　また、メタデータファイルの `parameters` で指定したパラメータを、アクション実行時のコールバックメソッド `run` の引数で受け取ることができます。  
　ここでは、入力パラメータ `context` で受けた値を設定ファイルで指定したログファイルに書き出す処理を行っています。  

### 動作確認  
　それでは、ここまでに作成したセンサ、トリガ、そしてアクションを動かします。  
　まずは `mypack` をデプロイします。`mypack` のリポジトリを取得して `make` コマンドを実行します。  

```sh
$ git clone https://github.com/userlocalhost2000/st2-pack-example.git
$ cd st2-pack-example
$ make
```
　
　ここでは、上記で作成した `mypack` をコンテンツディレクトリのパックが参照されるディレクトリ `/opt/stackstorm/packs/` にデプロイし、`mypack` の実行環境を構築するために `packs.setup_virtualenv` アクションを実行します。続いて、`st2ctrl reload` を実行することで、デプロイした `mypack` を読み込みます。  

　全ての処理が完了したら、以下のコマンドで正常にセンサ、トリガ、アクションが登録されたことを確認してください。  

```sh
vagrant@st2-node:~/st2-pack-example$ st2 sensor list --pack=mypack
+------------------------+--------+-----------------------------------------------+---------+
| ref                    | pack   | description                                   | enabled |
+------------------------+--------+-----------------------------------------------+---------+
| mypack.DirectorySensor | mypack | An example sensor to know how to implement it | True    |
+------------------------+--------+-----------------------------------------------+---------+
vagrant@st2-node:~/st2-pack-example$ st2 trigger list --pack=mypack
+---------------------+--------+---------------------------------------------+
| ref                 | pack   | description                                 |
+---------------------+--------+---------------------------------------------+
| mypack.changed_file | mypack | Trigger which indicates a change of trigger |
+---------------------+--------+---------------------------------------------+
vagrant@st2-node:~/st2-pack-example$ st2 action list --pack=mypack
+-----------------------+--------+----------------------------------------+
| ref                   | pack   | description                            |
+-----------------------+--------+----------------------------------------+
| mypack.output_context | mypack | An example action to know how it works |
+-----------------------+--------+----------------------------------------+
vagrant@st2-node:~/st2-pack-example$ 
```

　また、以下の通りセンサプロセスが正常に動いていることを確認してください。  

```sh
vagrant@st2-node:~/st2-pack-example$ ps aux | grep DirectorySensor
st2       7405  2.4  3.5 152892 73544 ?        S    03:57   1:36 /opt/stackstorm/virtualenvs/mypack/bin/python /opt/stackstorm/st2/local/lib/python2.7/site-packages/st2reactor/container/sensor_wrapper.py --pack=mypack --file-path=/opt/stackstorm/packs/mypack/sensors/directory_sensor.py --class-name=DirectorySensor --trigger-type-refs=mypack.changed_file --parent-args=["--config-file", "/etc/st2/st2.conf"] --poll-interval=30
vagrant   7807  0.0  0.0  10460   932 pts/0    S+   05:02   0:00 grep --color=auto DirectorySensor
vagrant@st2-node:~/st2-pack-example$ 
```

　上記が確認できましたら、最後に `mypack.changed_file` と `mypack.output_context` を紐づけるルール `mypack_test` を登録します。このルールは `mypack` のリポジトリの `rules` ディレクトリ以下にあります。  

```sh
vagrant@st2-node:~/st2-pack-example$ st2 rule create rules/mypack_test.yaml
+-------------+--------------------------------------------------------------+
| Property    | Value                                                        |
+-------------+--------------------------------------------------------------+
| id          | 583bbd3b1d41c81acce424ab                                     |
| name        | mypack_test                                                  |
| pack        | mypack                                                       |
| description | A test rule for testing mypack                               |
| action      | {                                                            |
|             |     "ref": "mypack.output_context",                          |
|             |     "parameters": {                                          |
|             |         "context": "[{{trigger.time}}] ({{trigger.status}})  |
|             | {{trigger.path}}"                                            |
|             |     }                                                        |
|             | }                                                            |
| criteria    |                                                              |
| enabled     | True                                                         |
| ref         | mypack.mypack_test                                           |
| tags        |                                                              |
| trigger     | {                                                            |
|             |     "type": "mypack.changed_file",                           |
|             |     "ref": "mypack.changed_file",                            |
|             |     "parameters": {}                                         |
|             | }                                                            |
| type        | {                                                            |
|             |     "ref": "standard",                                       |
|             |     "parameters": {}                                         |
|             | }                                                            |
| uid         | rule:mypack:mypack_test                                      |
+-------------+--------------------------------------------------------------+
vagrant@st2-node:~/st2-pack-example$ 
```

　ここまでで `mypack` を動かす全ての準備が整いました。それではセンサ `DirectorySensor` が監視するディレクトリに適当なファイルを作成します。  

```
vagrant@st2-node:~/st2-pack-example$ sudo touch /opt/stackstorm/hoge
```

　暫くすると `mypack` の設定ファイル `/opt/stackstorm/packs/mypack/config.yaml` の `log` パラメータで指定したパスに、トリガから送られた値が出力されていることがわかります。  

```
vagrant@st2-node:~/st2-pack-example$ tail -n2 /opt/stackstorm/packs/mypack/config.yaml 
# This parameter is used by output_context action
log: /tmp/output
vagrant@st2-node:~/st2-pack-example$ cat /tmp/output
[1480310219.11] (created) /opt/stackstorm/hoge
[1480310245.29] (created) /opt/stackstorm/packs/mypack/actions/output_context.pyc
vagrant@st2-node:~/st2-pack-example$ 
```

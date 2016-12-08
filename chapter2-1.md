## StackStorm の機能拡張
　前回の記事の [内部アーキテクチャ](https://github.com/userlocalhost/st2-draft/blob/master/chapter1-2.md) において、外部のサービスやシステムと連携して、処理やワークフローを実行する仕組みを解説しました。これらを実現するための内部コンポーネントであるセンサ、トリガ、アクションは、Pack という単位で管理されていたことを思い出してください。  

　StackStorm では、これらの Pack を動的に追加(削除)できる機能によって、柔軟な機能追加(削除)が行えます。また [StackStorm Exchange](https://exchange.stackstorm.org/) から、様々なサービス・システム向けの Pack が提供されており、これらを活用することで、運用をする上で必要となるおおよその機能はカバーできると思います。  

　しかし StackStorm Exchange から取得できる Pack に欲しい機能が実装されていなかったり、すでに利用している運用自動化の資産を活用したかったりする場合があるかもしれません。  
　ここでは、そうした運用のための開発を行うユーザ向けに、Pack 及び Pack を構成するセンサ、トリガ、アクションを実装する方法を解説します。  

　例として、ローカルマシンのファイルシステムのディレクトリの変更を監視するセンサ(`DirectorySensor`) とファイル更新が行われた際に引かれるトリガ(`changed_file`)、及び入力されたパラメータをログに吐き出すアクション(`output_context`) を実装します。  
　あまり面白みの無い仕組みですが、これらをどのように実装するのかを理解するにはちょうどよいと思います。  

### Pack を作る
　まずは [st2sdk](https://github.com/StackStorm/st2sdk) を使って Pack の雛形を作成します。以下の手順で `st2sdk` をインストールし、オリジナルの Pack (`mypack`) を作成します。  

```
$ sudo apt-get install python-pip   ## pip コマンドをインストール
$ sudo pip install st2sdk           ## st2sdk をインストール
$ st2sdk bootstrap mypack           ## pack の雛形を作成
```

　`st2sdk` のサブコマンド `bootstrap` によって、Pack 自体の定義ファイル `pack.conf` やセンサやアクションが参照するパラメータを保存する設定ファイル `config.yaml` など、最低限必要のファイルが自動生成されます。  
　なお、以降で解説するコードを含む `mypack` 全体を以下のリポジトリで公開しています。  

* [https://github.com/userlocalhost/st2-pack-example](https://github.com/userlocalhost/st2-pack-example)

### Sensor と Trigger を作る
　続いてセンサとトリガを作成します。コードを見る前にセンサに関するソフトウェアの構造について把握したいと思います。以下の図はセンサのソフトウェアアーキテクチャを表します。  

![Sensor のソフトウェアアーキテクチャ](https://raw.githubusercontent.com/userlocalhost/st2-draft/master/img/sensor-implementation.png)

　上部の 'mypack.DirectorySensor' と書かれた部分がユーザが実装する部分で、外部システムのイベントを検知する処理を実装します。SensorService は StackStorm の機能で、センサがトリガやデータストア、設定ファイルなどにアクセスするための仕組みを提供しています。具体的には、MongoDB で管理されるデータストアへのアクセスや、トリガを引いて RabbitMQ を通してイベント通知を送るためのインターフェイスを提供します。  
　また StackStorm は各センサごとにプロセスを立ち上げます。従って、ユーザが定義したセンサがクラッシュした場合でも、他のセンサや StackStorm サービスに対して影響を及ぼすことはありません。  

　ここから、センサを作成するコードを見て行きます。センサは、メタデータファイルとソースコードから構成されています。以下は冒頭のセンサのメタデータファイルの全文になります。  

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

　`trigger_types` 以下で、このセンサに紐付くトリガ (`changed_file`) を指定しています。その際、このトリガがどういった出力パラメータをアクションに渡すかを規定したフォーマット `payload_schema` を定義できます。センサはトリガを引く際、ここで規定したパラメータを設定します。ユーザが Rule を記述する際、ここで規定されているフォーマットに基づいて、パラメータの変換設定を行います。  
　`entry_point` ではセンサの実装ファイルを指定します。ここで指定するファイルのパスは、mypack のセンサに関連するファイルを格納するディレクトリ `/opt/stacksotm/packs/mypack/sensors` からの相対パスで指定します。また `class_name` で後述するセンサを実装したクラスを指定します。  

　それでは、センサの実装を見て行きます。以下はソースコード `directory_sensor.py` の抜粋になります。  

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

　センサの実装は大きく２種類あり、メールや Web プッシュ通知などの仕組みによって外部システムのイベントを受動的に受け取るパッシブセンサ `Sensor` か、センサ自体が外部システムを参照して能動的にイベントを取得しに行くアクティブセンサ `PollingSensor` のいづれかになります。  
　今回実装する `DirectorySensor` は定期的にファイルシステムを確認して差分を確認するため、アクティブセンサ `PollingSensor` を継承します。逆に [inotify](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/inotify.7.html) などの仕組みによって、ファイルシステムから送られてくる変更通知を待ち受けるセンサを実装する場合には、パッシブセンサ `Sensor` を継承すると良いです。  

　センサは、プロセスが実行された場合や初期化された場合など、幾つかのコールバックを受け取ることができます。ここでは、センサが初期化された後に１度だけ呼ばれるコールバックメソッド `setup` と、メタデータファイルの `poll_interval` パラメータで指定した間隔 (単位は秒) 毎に呼び出される `poll` メソッドに処理を記述します。  
　処理の中身は、まず初期化処理においてプライベートメソッド `_get_dirinfo` を呼び出し、以下に示す `mypack` の設定ファイルで指定したディレクトリ配下の各ファイルの更新時間を取得する処理 `_do_get_dirinfo` を実行し、結果を返します ([_do_get_dirinfo のコード](https://github.com/userlocalhost/st2-pack-example/blob/master/sensors/directory_sensor.py#L54-L60) の説明は割愛します)。  

```yaml
### /opt/stackstorm/packs/mypack/config.yaml
---
sensor:
    directories:
        - /opt/stackstorm
```

　続いて定期的に呼び出される `poll` において、同じように `_get_dirinfo` を呼び出し、前回実行時との差分 (作成、削除、更新されたファイル) を取得し、差分があった場合に `_dispatch_trigger` を呼び出し、トリガ (`mypack.changed_file`) を引きます。その際、メタデータファイルの `payload_schema` で規定したフォーマットに従って、アクションに渡すパラメータを設定します。  
　こうして通知されたイベントが MQ を経由し、`st2rulesengine` サービスに渡ります。そして、当該トリガに関連付けられたルールが存在する場合には、それに紐付くアクションがワーカノードで実行されます。  

### Action を作る
　最後にアクション (`output_context`) を作成します。アクションもメタデータファイルとソースコードの２つから構成されます。以下にメタデータファイルの全文を記載します。  

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

　メタデータファイルの構成は、[入門編の最後](https://github.com/userlocalhost/st2-draft/blob/master/chapter1-5.md) で解説した Workflow とほぼ同じです。ただ WorkFlow と異なり、このアクションは Python スクリプトで記述するために `runner_type` に `run-python` を指定します (他にも [様々な形式](https://docs.stackstorm.com/actions.html#action-runner) のアクションを指定できます)。  
　続いてアクションのソースコードの全文を以下に記載します。  

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
                return (False, "IOError occurred (%s)" % (err))

            return (True, "This processing succeeded.")
        else:
            return (False, "The output filepath is invalid.")
```

　当該アクションを実装したクラス `OutputContext` は、StackStorm が用意したベースクラス `Action` を継承しています。内部では、Pack の設定やデータストア、ログにアクセスするための仕組みを提供する `ActionService` オブジェクトへの参照を持っており、これらと連携したアクションを簡単に記述することができます。  
　今回実装するアクション `OutputContext` では、実行時に呼び出されるコールバックメソッド `run` をオーバーライドしています。メソッド `run` では、メタデータファイルの `parameters` で指定した値 `context` を仮引数で受け取ることができます。`run` の内部では、入力パラメータ `context` で受けた値を設定ファイルで指定したログファイルに書き出す処理を行っています。  

### 動作確認  
　それでは、ここまでに作成したセンサ、トリガ、そしてアクションを動かします。  
　まずは `mypack` をデプロイします。`mypack` のリポジトリを取得して `make` コマンドを実行します。  

```sh
vagrant@st2-node:~$ git clone https://github.com/userlocalhost/st2-pack-example.git
vagrant@st2-node:~$ cd st2-pack-example
vagrant@st2-node:~/st2-pack-example$ make
```
　
　ここでは、上記で作成した `mypack` を `/opt/stackstorm/packs/` にデプロイし、`mypack` の実行環境を構築するためにアクション(`packs.setup_virtualenv`) を実行します。続いて、`st2ctrl reload` を実行することで、デプロイした `mypack` の設定ファイルを読み込み、センサ (`DirectorySensor`) を起動させます。  

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
vagrant@st2-node:~/st2-pack-example$ st2 rule list --pack=mypack
+--------------------+--------+--------------------------------+---------+
| ref                | pack   | description                    | enabled |
+--------------------+--------+--------------------------------+---------+
| mypack.mypack_test | mypack | A test rule for testing mypack | True    |
+--------------------+--------+--------------------------------+---------+
vagrant@st2-node:~/st2-pack-example$ 
```

　また、以下の通りセンサプロセスが正常に動いていることも確認してください。  

```sh
vagrant@st2-node:~/st2-pack-example$ ps aux | grep DirectorySensor
st2       7405  2.4  3.5 152892 73544 ?        S    03:57   1:36 /opt/stackstorm/virtualenvs/mypack/bin/python /opt/stackstorm/st2/local/lib/python2.7/site-packages/st2reactor/container/sensor_wrapper.py --pack=mypack --file-path=/opt/stackstorm/packs/mypack/sensors/directory_sensor.py --class-name=DirectorySensor --trigger-type-refs=mypack.changed_file --parent-args=["--config-file", "/etc/st2/st2.conf"] --poll-interval=30
vagrant   7807  0.0  0.0  10460   932 pts/0    S+   05:02   0:00 grep --color=auto DirectorySensor
vagrant@st2-node:~/st2-pack-example$ 
```

　ここまでで `mypack` を動かす全ての準備が整いましたので、実際にこれらを動かしてみます。以下のように `DirectorySensor` が監視するディレクトリに適当なファイルを作成してください。  

```
vagrant@st2-node:~$ sudo touch /opt/stackstorm/packs/hoge
```

　暫くすると `mypack` の設定ファイル `/opt/stackstorm/packs/mypack/config.yaml` の `log` パラメータで指定したパスにトリガから送られた値が出力されます。出力先のファイルパスの確認と併せて、出力結果を確認します。  

```
vagrant@st2-node:~$ tail -n2 /opt/stackstorm/packs/mypack/config.yaml 
# This parameter is used by output_context action
log: /tmp/output
vagrant@st2-node:~$ cat /tmp/output
[1480310219.11] (created) /opt/stackstorm/hoge
[1480310245.29] (created) /opt/stackstorm/packs/mypack/actions/output_context.pyc
vagrant@st2-node:~$ 
```

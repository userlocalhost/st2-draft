## 動かしてみる
　構築した StackStorm 環境に対して、以下のように AWS で発生したイベントを検知してアクション `core.local` を実行する環境を構築します（'core.local' がどんな処理を行うかについては以降で解説します）。  

![構築する環境](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/picture2.png)

　ここでは [Amazon CloudWatch](https://aws.amazon.com/jp/cloudwatch/) で検出された [Amazon EC2](https://aws.amazon.com/jp/ec2/) のリソース変更のイベントを検知し、`core.local` アクションを実行させます。  
　core.local は Worker ノードにおいて任意のコマンドを実行するアクションになります。StackStorm では次のように、アクションを単体実行することもできます。

```
vagrant@st2-node:~$ st2 run core.local cmd=date
.
id: 583d31566f086f6556b9669c
status: succeeded
parameters: 
  cmd: date
result: 
  failed: false
  return_code: 0
  stderr: ''
  stdout: Tue Nov 29 07:42:14 UTC 2016
  succeeded: true
vagrant@st2-node:~$ 
```

  ワーカノードでアクション `core.local` を実行し、パラメータ `cmd` で受けたコマンド `date` を実行し、標準出力結果が上記の `result.stdout` から確認できます。また `status` から、当該アクションが正常終了したことが確認できます。
　以下では、これを冒頭の図が示すように 'aws.sqs_new_message' トリガが引かれた際に実行されるようにする方法を解説します。  

### 環境構築
　ここでは以下の設定を行い、AWS のイベントが呼ばれた際に core.local アクションが実行されるようにします。  

* aws pack のインストール
* aws pack の設定
* Amazon SQS, CloudWatch の設定
* Rule の設定

#### aws pack のインストール
　StackStorm は [st2contrib](https://github.com/StackStorm/st2contrib) というリポジトリでサードパーティの pack を提供しています。以下コマンドで aws pack をインストールします。  

```
$ st2 run packs.install packs=aws
```

#### aws pack の設定
　インストールされた pack に関連するファイル (コンテンツ) は、通常 `/opt/stackstorm/packs` 配下に展開されます。今回インストールした aws pack のコンテンツのディレクトリは `/opt/stackstorm/packs/aws` になります。  
　また各 pack の設定ファイル 'config.yaml' が、コンテンツディレクトリのトップに配置されます。ここで aws の設定ファイルを次のように編集してください。  

```yaml
---
aws_access_key_id: "*****"
aws_secret_access_key: "******"
region: "ap-northeast-1"
interval: 20
st2_user_data: "/opt/stackstorm/packs/aws/actions/scripts/bootstrap_user.sh"

service_notifications_sensor:
  host: "localhost"
  port: 12345
  path: "/my-path"

sqs_sensor:
  input_queues: "notification_queue"

sqs_other:
  max_number_of_messages: 1
```

　コメントにあるように 'setup' キー以下の項目は下位互換のために残されている項目なので、ここでは削除します。  
　次に、アクセスキー (アクセスキーID / 秘密アクセスキー) とリージョンを設定します。アクセスキーの取得方法については [AWS のドキュメント](http://docs.aws.amazon.com/ja_jp/AWSSimpleQueueService/latest/SQSGettingStartedGuide/AWSCredentials.html) を参照してください。リージョンは東京リージョン 'ap-northeast-1' を指定します。  
　最後に、イベントメッセージが通知される [Amazon SQS](https://aws.amazon.com/jp/sqs/) のキュー名 'notification_queue' を指定します。ここで指定したキューはこの後で AWS マネジメントコンソールから作成します。  

#### AmazonSQS, CloudWatch の設定
　AmazonSQS は Amazon が提供するフルマネージの MQ サービスで、CloudWatch はモニタリングサービスです。CloudWatch において、EC2 や S3 のリソースが変更された際に、AmazonSQS に通知メッセージを送るように設定します。  
　StackStorm は SQS に送られたメッセージを取得することで、間接的に AWS のイベントハンドリングを行うことができます。StackStorm は SQS を利用する方法の他に、[Amazon SNS](https://aws.amazon.com/jp/sns/) を利用して通知を取得する仕組みも提供しています。後者の設定方法については、本稿では渇愛します。  

　では、それぞれの設定を実施します。  
　まず以下のように AmazonSQS に先ほどの設定ファイル `config.yaml` の `sqs_sensor.input_queues` で指定した名前のキューを作成します。  

![AmazonSQS でキューを作成](sqs.png)

　続いて CloudWatch で EC2 のイベントを SQS のキュー 'notification_queue' に送るルールを行います。  

![EC2 のイベントを SQS に通知](cloudwatch_config.png)

　ここまでの設定で EC2 のイベントを aws pack のセンサ AWSSQSSensor で検知する準備が整いました。  
　
#### Rule の設定
　最後に、センサ AWSSQSSensor がトリガ aws.sqs_new_message を引いた際に、アクション core.local を実行するルールを定義します。  
　以下がルール定義の全文になります。ルール定義は YAML 形式で記述します。以下のルールを記述するファイル名は任意の名前で問題ありません。ここでは 'ec2_event_handling.yaml' とつけました。  

```yaml
---
name: 'ec2_event_handling'
description: 'A rule to get events on Amazon EC2'
enabled: True
trigger:
  type: aws.sqs_new_message

action:
  ref: 'core.local'
  parameters:
    cmd: 'echo "{{trigger.body}}" >> /tmp/results'

criteria:
  trigger.queue:
    pattern: 'notification_queue'
    type: 'equals'
```

　2,3 行目の `name` と `description` は、それぞれルールのラベルと説明文を表します。上記のルールを登録したあとにルールの一覧を表示する `st2 rule list` コマンドを実行すると以下のように表示されます。  

```
vagrant@st2-node:~$ st2 rule list
+----------------------------+---------+------------------------------------+---------+
| ref                        | pack    | description                        | enabled |
+----------------------------+---------+------------------------------------+---------+
| default.ec2_event_handling | default | A rule to get events on Amazon EC2 | True    |
+----------------------------+---------+------------------------------------+---------+
vagrant@st2-node:~$ 
```

　- 行目の trigger と - 行目の action パラメータによって、トリガとアクションの紐付けを行っています。それぞれ - 行目の `trigger.type` と - 行目の `action.ref` パラメータでどのトリガが引かれたら、どのアクションを実行するかを指定しています。  
　また - 行目の `action.parameters` で、トリガから渡されるパラメータとアクションのパラメータの変換設定を行っています。トリガとアクションがそれぞれがどういったパラメータを持っているかについては `st2 action get` と `st2 trigger get`コマンドで確認できます。以下は、それぞれのパラメータを確認した結果を表します。  

```
vagrant@st2-node:~$ st2 trigger get aws.sqs_new_message
+-------------------+--------------------------------------------------------+
| Property          | Value                                                  |
+-------------------+--------------------------------------------------------+
| id                | 582ff9c388fa3f318505cb08                               |
| ref               | aws.sqs_new_message                                    |
| pack              | aws                                                    |
| name              | sqs_new_message                                        |
| description       | Trigger which indicates that a new message has arrived |
| parameters_schema |                                                        |
| payload_schema    | {                                                      |
|                   |     "type": "object",                                  |
|                   |     "properties": {                                    |
|                   |         "queue": {                                     |
|                   |             "type": "string"                           |
|                   |         },                                             |
|                   |         "body": {                                      |
|                   |             "type": "object"                           |
|                   |         }                                              |
|                   |     }                                                  |
|                   | }                                                      |
| tags              |                                                        |
| uid               | trigger_type:aws:sqs_new_message                       |
+-------------------+--------------------------------------------------------+
vagrant@st2-node:~$
```

```
vagrant@st2-node:~$ st2 action get core.local
+-------------+--------------------------------------------------------------+
| Property    | Value                                                        |
+-------------+--------------------------------------------------------------+
| id          | 582513ef88fa3f5072eee96c                                     |
| uid         | action:core:local                                            |
| ref         | core.local                                                   |
| pack        | core                                                         |
| name        | local                                                        |
| description | Action that executes an arbitrary Linux command on the       |
|             | localhost.                                                   |
| enabled     | True                                                         |
| entry_point |                                                              |
| runner_type | local-shell-cmd                                              |
| parameters  | {                                                            |
|             |     "cmd": {                                                 |
|             |         "required": true,                                    |
|             |         "type": "string",                                    |
|             |         "description": "Arbitrary Linux command to be        |
|             | executed on the remote host(s)."                             |
|             |     },                                                       |
|             |     "sudo": {                                                |
|             |         "immutable": true                                    |
|             |     }                                                        |
|             | }                                                            |
| notify      |                                                              |
| tags        |                                                              |
+-------------+--------------------------------------------------------------+
vagrant@st2-node:~$ 
```

　トリガ `aws.sqs_new_message` の出力として `queue` と `body` のパラメータを持つオブジェクトが得られ、アクション `core.local` の入力として `cmd` と `sudo` パラメータがあることがわかります。またアクションの入力パラメータのうち `cmd` は `required` が `True` となっており、必須パラメータであることを表しています。先に示したルールでは [Jinja Template](http://jinja.pocoo.org/docs/dev/templates/) で記述することで、トリガの出力から `body` パラメータを取り出し、アクションの `cmd` パラメータに渡しています。  

　最後に - 行目の `enabled` パラメータで、当該ルールを有効化するかどうかを設定できます。False を指定した場合には、ルールで指定したトリガが引かれてもアクションを実行しません。  

### 動作確認
 ここでいよいよルールを登録して EC2 にインスタンスを作成し、そのイベントを検知できるか確認します。  
 まずは以下のコマンドでルールを登録します。  

```
vagrant@st2-node:~$ ls -l ec2_event_handling.yaml
-rw-rw-r-- 1 vagrant vagrant 307 Nov 22 05:43 ec2_event_handling.yaml
vagrant@st2-node:~$ st2 rule create ./ec2_event_handling.yaml
+-------------+----------------------------------------------------------+
| Property    | Value                                                    |
+-------------+----------------------------------------------------------+
| id          | 5833dba788fa3f6570b3a27b                                 |
| name        | ec2_event_handling                                       |
| pack        | default                                                  |
| description | A rule to get events on Amazon EC2                       |
| action      | {                                                        |
|             |     "ref": "core.local",                                 |
|             |     "parameters": {                                      |
|             |         "cmd": "echo "{{trigger.body}}" >> /tmp/results" |
|             |     }                                                    |
|             | }                                                        |
| criteria    | {                                                        |
|             |     "trigger.queue": {                                   |
|             |         "pattern": "notification_queue",                 |
|             |         "type": "equals"                                 |
|             |     }                                                    |
|             | }                                                        |
| enabled     | True                                                     |
| ref         | default.ec2_event_handling                               |
| tags        |                                                          |
| trigger     | {                                                        |
|             |     "type": "aws.sqs_new_message",                       |
|             |     "ref": "aws.sqs_new_message",                        |
|             |     "parameters": {}                                     |
|             | }                                                        |
| type        | {                                                        |
|             |     "ref": "standard",                                   |
|             |     "parameters": {}                                     |
|             | }                                                        |
| uid         | rule:default:ec2_event_handling                          |
+-------------+----------------------------------------------------------+
vagrant@st2-node:~$ 
```

　続いて EC2 のマネジメントコンソールにログインし、インスタンス 'test-instance' を作成します。  

![EC2 にインスタンスを作成](ec2_create_instance.png)

　程なくして、ルールで記述したファイル `/tmp/results` に CloudWatch の出力が表示されることが確認できます。  

```
vagrant@st2-node:~$ ls -lt /tmp/results 
-rw-r--r-- 1 stanley stanley 610 Nov 22 06:37 /tmp/results
vagrant@st2-node:~$ cat /tmp/results 
{version:0,id:ac2c2f0f-e969-4cc4-8578-d64af974cce6,detail-type:EC2 Instance State-change Notification,source:aws.ec2,account:353232136563,time:2016-11-22T06:36:47Z,region:ap-northeast-1,resources:[arn:aws:ec2:ap-northeast-1:353232136563:instance/i-6a8547f4],detail:{instance-id:i-6a8547f4,state:pending}}
{version:0,id:d7a5ad4e-18e7-4fef-9119-4df9e905faef,detail-type:EC2 Instance State-change Notification,source:aws.ec2,account:353232136563,time:2016-11-22T06:36:57Z,region:ap-northeast-1,resources:[arn:aws:ec2:ap-northeast-1:353232136563:instance/i-6a8547f4],detail:{instance-id:i-6a8547f4,state:running}}
vagrant@st2-node:~$ 
```

　ここまで EC2 のイベントを StackStorm で受け取り、ユーザ定義のルールに沿って処理する方法について解説してきました。ここまでの内容を応用することで、様々なアプリケーションサービスと連携する処理を簡単に設定することができるようになります。  

　尚、本章の内容を実際に動かして確認するにはクレジットカード登録を行った AWS アカウントが必要になり、手軽に試すことができないユーザもいるかもしれません。そうした方は [ツチノコブログ](http://tsuchinoko.dmmlabs.com/?p=4623) に別の例 (`GitHub` と `RabbitMQ` を利用したイベントハンドリング) を示しましたので、こちらをご参照ください。  

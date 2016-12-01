## 内部アーキテクチャ
　ここでは、実際に StackStorm を使ってみようと思っている方向けに、StackStorm がどのような仕組みでどのように動作するかについて解説します。  

　以下は、StackStorm がイベントを処理する一連の処理の流れを表した図になります。  

![イベント処理の流れ](https://raw.githubusercontent.com/userlocalhost2000/st2-draft/master/img/picture1.png)

　StackStorm の IFTTT x Workflow 処理の内部は Sensor, Trigger そして Action(Workflow) の３つの要素から構成されています。そして、これら３つの要素を機能毎にまとめたソフトウェアモジュールのことを `pack` と呼びます。StackStorm では動的に `pack` を追加・削除できる拡張性の高い機構になっています。  
　ここでは pack の内部コンポーネントがどのような役割でどのように動作するかについて解説します。  

　まず Sensor は外部で発生したイベントを監視するプロセスで、外部のイベントを検知した際には、紐付けられた Trigger にイベントを通知 (以降では "Trigger を引く" と表現) します。  
　Action は、`HTTP リクエストを送る` や `メールを送る` といった特定の目的に特化した処理を行うプロセスで Trigger に紐付けられます。そして Trigger は Sensor から通知された外部イベントを Action が扱い易い形に正規化します。Trigger に紐付く Action が何も設定されていない状態では、Sensor が Trigger を引いても何も起こりません。  

　ここで各要素の関連について、もう少し詳しく解説します。  
　Sensor と Trigger は静的に紐付けられている (図の青枠) のに対して、Trigger と Action はユーザによって動的に紐付けられます (図の緑枠)。この Trigger と Action を紐付ける仕組みのことを `Rule` と呼びます。  
　Rule には二つの役割があります。一つは上述した Trigger と Action/Workflow を紐づける役割で、もう一つが Trigger パラメータと Action パラメータを変換する役割です。  
　StackStorm では Trigger と Action はそれぞれ独立しており、Rule によって任意に Trigger と Action を組み合わせることができます。その際、Trigger から渡されるどのパラメータを抽出し、どういったパラメータを Action に渡すかの設定を Rule に記述します。  

　最後に Action と併記されている Workflow について解説します。  
　Workflow は複数の Action を記述できる Action になります。Action と同じように、Rule に紐付けられた Trigger が引かれた際に実行されます。Action と Workflow の違いは、Action が特定の目的に特化した処理を記述した 'プログラムコード' であるのに対して、Workflow は実行する１つ以上の Action と、それらの依存関係および各 Action の実行結果に応じた処理の流れを記述した '定義' になります。  
　ユーザはプログラムコードを記述せずに、 Action の実行順序や処理の流れの定義した Workflow を記述することで、複数のアプリケーションサービスを跨いだ複雑な処理を実現できます。また Trigger と紐付けた Rule を記述することで、それらを自動化させることができます。 

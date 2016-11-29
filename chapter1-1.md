## StackStorm の特徴
　ここでは StackStorm が、単に依存関係を解決しながら処理を実行するワークフローエンジンと何が違うかについて、機能面と運用面から見て行きます。  

### 機能面
　機能的な StackStorm の特徴として、ワークフロー処理をイベントドリブンに行うことが挙げられます。  
　例えば、先ほどの処理をタスク管理システムのチケットが発行されたことを検知して処理をすることや、パブリッククラウド管理下のリソースやソースコード管理システムのコンテンツの変更を検知して、ワークフロー処理を実行させるといったことが実現できます。  

　もちろん、こうしたイベントドリブンな運用の自動化は StackStorm が無くとも実現できます。  
　例えば git のローカルリポジトリに変更が加えられた際に、何らかの処理を実行したいといった場合には [Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) を使えばいいでしょうし、GitHub のリポジトリに対する PullRequest や Issue の作成などを検知したい場合には、[GitHub が提供している Webhook の仕組み](https://developer.github.com/webhooks/) が便利です。また [AWS の Lambda](http://docs.aws.amazon.com/lambda/latest/dg/welcome.html) を利用することで AWS の各種サービスのイベントを検知することもできます。  
　このように、それぞれのソフトウェア、サービスが提供する仕組みを個別に利用することで、それぞれのイベントを検知できます。しかし、各サービス毎の連携を実現させるための仕組みを個別に実装をするのは、骨が折れます。  

　これに対して、StackStorm は様々なサービス間の連携を行うことを目的とした仕組み ([IFTTT](https://ifttt.com/)) の考え方を、ワークフローエンジンに取り入れました。  
　結果 StackStorm を使うユーザは、どういったサービスからどのようなイベントに対して、どのような処理をするかというワークフローを記述してやるだけで、実際にイベントを検知する仕組みや、外部アプリケーションに対する処理は StackStorm が良しなに実行してくれます。  
　[こちら](https://github.com/StackStorm/st2contrib#available-packs) が StackStorm で利用可能なアプリケーション (サービス) の一覧になります。  

### 運用面
　StackStorm 自体が Scalable かつ High Available なアーキテクチャであるということが運用面での特徴として挙げられます。  
　以下は StackStorm のアーキテクチャを表しています。内部アーキテクチャについてはこの後解説しますが、ここではどの点が Scalable で High Available なのかを簡単に解説します。  

![StackStorm architecture diagram](https://docs.stackstorm.com/_images/architecture_diagram.jpg)
(出典：[StackStorm Overview - How it Works](https://docs.stackstorm.com/overview.html#how-it-works))

　まず上段の "StackStorm Sensors" というラベルの書かれた四角に注目してください。StackStorm ではこれらのノードで外部サービスのイベントを検知し、複数台で協調して動作する仕組みを持っており、システムを High Available にすることができます。  
　また、通知されたイベントを処理するノード (中段の "StackStorm Workers" と書かれた四角で囲まれたノード) 群は状態を持たないためスケールさせることができます。  

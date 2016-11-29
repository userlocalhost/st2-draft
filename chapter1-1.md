## StackStorm の特徴
　ここでは StackStorm が、単に依存関係を解決しながら処理を実行するワークフローエンジンと何が違うかについて、機能面と運用面から見て行きます。  

### 機能面
　機能的な StackStorm の特徴として、ワークフロー処理をイベントドリブンに行うことが挙げられます。  

　例えば、あなたが何がしかのサービスプロバイダーの運用担当者であったとして、サーバを構築してサービスインする作業を想像してみてください。  
　まずマシンリソースを調達し、調達したマシンに対してプロビジョニングを行うと思います。そして DNS への登録や LB への追加作業を行い、場合によってはこれらに加えて、リソース管理システムへの登録や、タスク管理システムへの記録などの管理・報告作業も発生するかもしれません。  
　StackStorm では、こうした処理をタスク管理システムのチケットが発行されたことを検知して処理をすることや、パブリッククラウド管理下のリソースやソースコード管理システムのコンテンツの変更を検知して、ワークフロー処理を実行させるといったことが実現できます。  

　もちろん、こうしたイベントドリブンな運用の自動化は StackStorm が無くとも実現できます。  
　例えば git のローカルリポジトリに変更が加えられた際に、何らかの処理を実行したいといった場合には [Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) を使えば実現できます。また GitHub のリポジトリに対する PullRequest や Issue の作成などを検知したい場合には、[GitHub が提供している Webhook の仕組み](https://developer.github.com/webhooks/) が便利です。更に [AWS の Lambda](http://docs.aws.amazon.com/lambda/latest/dg/welcome.html) や [Amazon SNS](https://aws.amazon.com/sns/) を利用することで AWS の各種サービスのイベントを検知することもできます。  
　このように、それぞれのソフトウェア、サービスが提供する仕組みを個別に利用することで、それぞれのイベントを検知できます。  
　しかし、各サービス毎の連携を実現させるための仕組みを個別に実装をするのは、骨が折れます。  

　これに対して、StackStorm は様々なサービス間の連携を行うことを目的とした仕組み ([IFTTT](https://ifttt.com/)) の考え方をワークフローエンジンに取り入れました。  
　結果 StackStorm を使うユーザは、どういったサービスからどのようなイベントに対して、どのような処理を行うかというワークフローを記述してやるだけで、実際にイベントを検知する仕組みや、外部アプリケーションに対する処理を実装せずにすみます。  

### 運用面
　運用面での特徴として、StackStorm 自体が Scalable かつ High Available なアーキテクチャであるということが挙げられます。  
　以下は StackStorm のアーキテクチャを表しています。内部アーキテクチャについてはこの後解説しますが、ここではどの点が Scalable で High Available なのかを簡単に解説します。  

![StackStorm architecture diagram](https://docs.stackstorm.com/_images/architecture_diagram.jpg)
(出典：[StackStorm Overview - How it Works](https://docs.stackstorm.com/overview.html#how-it-works))

　まず上段の "StackStorm Sensors" というラベルの書かれた四角に注目してください。StackStorm ではこれらのノードで外部サービスのイベントを検知し、複数台で協調して動作する仕組みによって、システムを High Available にすることができます。  
　また、通知されたイベントを処理するノード (中段の "StackStorm Workers" と書かれた四角で囲まれたノード) 群は状態を持たないためスケールさせることができます。これらの内部の詳しい仕組みについては、この後で詳しく解説します。  

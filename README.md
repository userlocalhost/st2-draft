# 初回原稿

## タイトル：StackStorm で始める運用自動化 ~入門編~

　運用自動化は、情報通信技術が民生化されて以降、現在にかけて挑戦され続けてきた大テーマです。  
　一口に自動化といっても様々な切り口が存在します。例えば、ソフトウェアのビルドやマシンリソースの調達・環境構築、または監視や通知の仕組み、あるいは CI など様々な分野で自動化のソリューションが提案され発展を遂げてきました。  
　本稿は一般に "ワークフロー" と呼ばれる予め規定された業務プロセスフローの処理を、入力や状況に応じて自動化するソリューションのひとつ [StackStorm](https://stackstorm.com/) について、２回に分けて詳しく解説します。  

　初回は、StackStorm を初めて知る人・使う人向けに、StackStorm がどのようなもので、どのような特徴を持って、どのように使うかについて解説して行きます。具体的に AWS と連携したイベントドリブンな自動化処理の設定の仕方や、細かなタスクを組み合わせた複雑なワークフローの設定方法を実際に動かしながら解説して行きます。  
　今回の内容によって、StackStorm の機能を一通り触ることで、おおよそこれがどんなものかわかってもらえると思います。  

* [StackStorm の特徴](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter1-1.md)
* [内部アーキテクチャ](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter1-2.md)
* [StackStorm をインストール](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter1-3.md)
* [StackStorm を動かしてみる](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter1-4.md)
* [ワークフローを記述する](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter1-5.md)
* [おわりに](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter1-6.md)

---
# 次回原稿
## タイトル：StackStorm で始める運用自動化 ~応用編~
　このエントリでは、イベントドリブンなワークフロー自動化ツール StackStorm について、２回に分けてたっぷりと解説してゆきます。  
　前回は、StackStorm を初めて知人、まだ使ったことが無い人向けに StackStorm がどんなもので、どのように使うかといったユーザ視点で解説してきました。  

　今回は、運用自動化を行う開発者、及びシステムの運用者に向けて、StackStorm に独自の拡張機能を追加する方法、及び StackStorm の運用のポイントについて解説します。運用については特に、高可用な環境構築の方法について詳しく解説します。  
　本エントリの内容を最後まで読んでいただくことで、StackStorm を用いた開発・運用に関する一通りの知識を習得できるようになると思います。  

* [StackStorm の機能拡張](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter2-1.md)
* [High Available な環境構築](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter2-2.md)
* [冗長構成な StackStorm の運用の注意点](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter2-3.md)
* [終わりに](https://github.com/userlocalhost2000/st2-draft/blob/master/chapter2-4.md)

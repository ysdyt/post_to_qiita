TwilioAPI×Python(Flask)×Herokuで「祝電2.0」アプリを作る

# はじめに

![ipod_shuffle](/Users/ysdyt/Downloads/IMG_9330.JPG)

新郎新婦には内緒で「ご結婚おめでとう」の「声」を友人から集めてプレゼントするという、「祝電2.0」的なアプリを作成しました。元ネタは2013年のコチラの素敵なブログ

- [「ご結婚おめでとう」親友に贈ったコードとデザインの話](http://mamipeko.hatenablog.com/entry/happy-wedding-s)


読まれたことが無い方は是非一読して頂きたいです。「プログラマーってほんとにかっこいいな」と思えます。このブログに痛く感動したので、こちらの内容を完コピして作成しました。

ブログ内容を簡単に要約すると、

- Twilioという音声APIを使って自動応答メッセージの電話アプリを作成
- 新郎新婦友人に指定の電話番号に電話してもらい、「結婚おめでとう」のメッセージを録音してもらう
- iPodなどにお祝いのメッセージを入れてプレゼント

ブログの内容と今回の差分としては、

- 元ネタ: Rubyで実装 → 今回: Python(Flask)で実装
- 元ネタ: twilioAPIで取得したデータをハンドリングするためにいい感じにGUIページを作成 → 今回: GUIは作成せず、twilioAPIの管理画面ページでデータ管理（手抜きした）

上記のブログや企画メンバーのエンジニアの方のブログには処理フローに関する言及はされているものの、動く形の完全なコードは公開されていなかったので、ブログ内容を参考にしながら同じ動作をするものをPythonで実装したという感じです。断片的なスクリプトはエンジニアメンバーの方がブログで公開されていました。

- [#wedding-sに向けてTwilioでお祝いのメッセージを集めた](http://hmsk.hatenablog.com/entry/2013/10/22/091427)

自分は普段はPythonを使ってデータ分析する仕事をしているので、Flaskのようなウェブフレームワークを使ってアプリっぽいものを作ったこともHerokuのようなホスティングサービスを使ったこともありませんでした。

そんなわけで、webアプリ（？）素人がいろいろコケた点や、TwilioAPI公式ドキュメントでは詳しく解説されていない点についてメモとして残します。


# TwilioAPIへの登録

Twilioの登録・課金方法・電話番号の購入・静的サンプルファイルのS3への設置と挙動確認などは以下のブログが参考になりました。今回は静的ファイルを扱わないのでスルーしてもok。ひとまずAPIへの登録が完了すれば大丈夫です。

- [電話API Twilioの基本的な使い方を調べてみた](http://dev.classmethod.jp/server-side/twilio/twilio-basic/)

Twilioは有料のAPIサービス（無料枠は無し）で、まず2000円をミニマム金額として課金し、以後1000円単位で追加課金する仕組み。1通話が0.1~2円程度なので、簡単なアプリ開発などなら最初の2000円で十分量だと思います。


ちなみに、「S3って何？」って場合はこちらページが参考になります。

- [AWS再入門 Amazon S3編](http://dev.classmethod.jp/cloud/aws/cm-advent-calendar-2015-aws-re-entering-s3/)
 
S3は、Amazonが運営するクラウドのファイルサーバーです。最初のクレジットカード登録などは面倒ですが、非常にお安く簡単に利用できるので使いましょう。（S3には自動電話案内のための音声データ置き場になります）


# ローカル環境の作成

- [PYTHONとFlask開発環境の設定方法](https://jp.twilio.com/docs/guides/how-to-set-up-your-python-and-Flask-development-environment#choose-a-python-version)


上記のページに従ってローカルの開発環境を整えます。主にやることは、以下のインストールです。

- pythonのインストール
- vertualenvのインストール
- Flaskのインストール
- Twilio Python SDKのインストール
- ngrokのインストール（後述）

vertualenvは仮想環境の構築を行うものです。webアプリのスクリプト一式をHerokuサーバーにホスティングするときに、不必要なモジュールなどが紛れ込まないようにvertualenvで今回のアプリ用に環境を切り離しておきます。後で面倒なことにならないようにインストールしておきましょう（anacondaのパッケージで仮想環境を作りたい場合は後述を参照）
ngrokについては詳細を後述しています。

# TwilioAPIの理解

- [VOICE TWIML PYTHON クイックスタート](https://jp.twilio.com/docs/quickstart/python/twiml#twilio-markup-language)

開発環境の設定が終わったら上記のクイックスタートの内容を眺めてみて、TwilioAPIで実際にどういうタスクがどんな感じで書けるのか確認します。（自分は実際に動かしてみてどんな感じで挙動するか逐一確かめないと理解できないタイプなので...）

Twilioは電話がかかってくると指定したURLにTwiML(XML的なもの)を見に来てくれて、そこで定義されているフォーマットの処理を随時実行していってくれます。

上記のチュートリアルをみてみると、TwiMLの書き方や動的な生成の仕方がわかりますが、一部古い記述（最新のAPI仕様に合っていない記述）があったり、肝心のwebサーバーにFlaskスクリプトをホストする方法が不親切だったりするのでいろいろコケます。。。
このチュートリアルでは「TwilioAPIの書き方ってだいたいこういう雰囲気なのね」ということを確認する程度でも良いと思います。

今回作成するアプリでは、TwilioAPIの機能のうち、主に着信通話のハンドリングに当たる「電話を受け付ける」「録音済みの音声ファイルを流す」「ユーザからのメッセージを録音する」「番号の入力を受け付ける」などを利用しています。

実際のアプリのシーケンスは以下のような流れです（[hmskさんのブログ](http://hmsk.hatenablog.com/entry/2013/10/22/091427)から引用）

- (1) 電話を受け付けてウェルカムなメッセージを流す
- (2) メッセージ入力方法の説明を流す（終わったら#押してねみたいな）
- (3) 発信音を鳴らして録音を実施する（#で終了する）
- (4) 録音したメッセージを流すよと宣言して流す
- (5) このメッセージで良かったら"1"を押せ、もう一度録るなら"3"を押せと促しつつ入力を待つ
- (6) 入力された数字が、"1"なら(7)へ、"3"なら(2)へ、それ以外なら(5)へリダイレクト
- (7) お礼を流して電話を切る


以下が各機能のチュートリアルとなりますが、上で書いた通り、チュートリアル通りに書いても動かない箇所があるので注意してください。

- [PYTHON クイックスタート：メッセージを読み上げる](https://jp.twilio.com/docs/quickstart/python/twiml/say-response#sample-app)
- [PYTHON クイックスタート: 発信者にMP3ファイルを再生する](https://jp.twilio.com/docs/quickstart/python/twiml/play-mp3-for-caller)
- [PYTHON クイックスタート: 発信者からのメッセージを録音する](https://jp.twilio.com/docs/quickstart/python/twiml/record-caller-leave-message)

上記のシーケンスと、チュートリアル、最新APIの使い方を参考にしながら書いた完成版のFlaskスクリプトを以下に置いておきます

- [https://github.com/ysdyt/twilio_wedding](https://github.com/ysdyt/twilio_wedding)

（スクリプト中で呼び出しているmp3ファイルは、S3に設置した自動応答用の音声データです。）



# ローカル環境での実行（ngrokの使い方）

次に、作成したwebアプリ（Flaskスクリプト）が正しく動くかローカルでテストをします。
面倒なのでデプロイ（サーバにアップ）することなく挙動をテストしたいですが、そんなときは`ngrok`が便利です。無料なので是非使いましょう。

- [テストサーバーへのアップが面倒なときはngrokでローカル環境を外部公開してみよう](https://liginc.co.jp/web/programming/156484)

<br>

OSに合ったngrokを[公式ページ](https://ngrok.com/)からDLします。Macなら`brew install ngrok`でok

DLしたファイルを解凍します

```bash
$ unzip /path/to/ngrok.zip
```

その後、`./negrok help`などで表示が確認されればok

`ngrok`を実際に使うためには、[ngrok公式ページ](https://ngrok.com/)からsign upでユーザー登録する必要があります（無料）

ユーザ登録してログインした後、[Get Started](https://dashboard.ngrok.com/get-started)の画面で表示されている`./ngrok authtoken (文字列が続く)` をターミナルで実行しておきます
（もしかしたら、このユーザ登録とauthtokenの登録は必須ではないかも）

ngrokの操作が一通り完了したら、動かしたいFlaskスクリプトの.pyを実行します
（実行すると localhost:5000 に表示できる状態になる）

Flaskが動いている状態で、別ウィンドウで`./ngrok http 5000` を実行します。

すると、新しく立ち上がる画面に

```bash
Forwarding   http://6814caa9.ngrok.io -> localhost:5000
Forwarding   https://6814caa9.ngrok.io -> localhost:5000
```

みたいなものが書かれているので、（httpでもhttpsの方でもいいが）表示されたURLをtwilioAPIの管理画面の「Webhook」のところに登録して保存します。
これで、ローカルで実行した内容がウェブサーバーで実行されているかのように処理されるようになります。

その後、当該の電話番号に電話するとFlaskで書かれた処理が実行されるはず。


ちなみに、ngrokは起動する度にランダムなurlを返します。もしこれを固定にしたい場合は[このページ](https://liginc.co.jp/web/programming/156484)の「ランダム文字列が面倒くさい」の項目を参照してください。
（※しかし、2017年4月段階では固定URLを取得するためには有料会員にならないといけなくなっているかも？）

# webサーバーへのホスト（Herokuの使い方）

静的な挙動（例えば、電話がかかってきたら指定のmp3を自動再生するだけ）をするためのTwiMLを作成するpythonスクリプトであれば、それをS3に置いて、スクリプトのURLをTwilioAPIの管理ページに登録すれば実行できます。

しかし、ユーザが入力する番号によって処理を変えたいなどの（今回のような）複雑な処理を行いたい場合は、外部のサーバーにpythonスクリプトを設置し、ユーザの入力内容に応じて動的にTwiMLが生成されるようにしなければいけません。

スクリプトを設置する外部サーバの選択肢として、Amazon Web Service (AWS)のEC2やGoogle Cloud Platform (GCP)のGoogle App Engine (GAE)などが低料金で使用可能ですが、今回は目的とする処理が単純ということもあり、設置も簡単で、かつ無料で利用できる [Heroku](https://www.heroku.com/)を利用することにしました。
[Heroku](https://www.heroku.com/)は、gitで管理されたwebスクリプト一式を指定のHerokuサーバにpushするだけで、アプリを動かすことができるありがたいサービスです。

pythonで書かれたwebアプリケーション（DjangoやFlask）をHeroku上で実行するための手順は以下の公式チュートリアルを参考にしました。

- [Getting Started on Heroku with Python](https://devcenter.heroku.com/articles/getting-started-with-python#introduction)
- （上記の内容の日本語版的なものが[Qiita](http://qiita.com/cfiken/items/0715bb945389bc9ca682)にもあった）

チュートリアルで行われている大まかな流れは、

- Herokuアカウントの作成
- Heroku Command Line Interface (CLI)のDL
- CLIを使ってherokuへのログイン
- `heroku create` でHeroku上にアプリ用のリポジトリを作成
- 作成したレポジトリに`git push heroku master`でスクリプトをpush
- `heroku open`でアプリの起動

という感じです。お手軽。
ちなみに、今回作成するアプリではDB系の操作を行わないので、SQLAlchemyやpsycopg2などのインストール処理などは全てぶっ飛ばしています。

<br>

アプリを動かすためにHerokuにpushしなければならない最低限のファイルは以下の３つです。

- main.py ・・・ Flaskのスクリプトが書かれた本体ファイル（ここでは仮にmain.pyというファイル名にしているだけ）
- Procfile ・・・ main.pyを実行するためのコマンドが書かれたファイル（詳細は後述）
- requirements.txt ・・・ main.pyでimportされるモジュールのリストファイル


（注意）以下の処理はvertualenvやanacondaでアプリ用に切り分けた仮想環境に入ってから操作します。


Herokuアカウントを登録します  
[http://www.heroku.com/](http://www.heroku.com/)

[ここ](https://devcenter.heroku.com/articles/getting-started-with-python#set-up)からOSに合ったHerokuのCLIツールをインストールします。

ユーザ登録とCLIインストール後、ターミナルで`heroku login`します

```
% heroku login 
#-> herokuのIDパスワードを入力
```

Herokuサーバーにデプロイしたいpythonスクリプト（Flaskのスクリプト）が格納されたディレクトリをgitの管理下に置きます

```
$ cd /path/to/dir
$ git init
```

同じディレクトリ下で、先にrequirements.txtとProcfileを作っておきます

```
$ pip freeze > requirements.txt
```
（もしも gunicornをインストールしていなければ `pip install gunicorn`する）
（"main"の部分はFlaskのファイル名に合わせる）

```
$ touch Procfile
$ echo web: gunicorn main:app --log-file - >> Procfile
```

`git init`した（同じ）ディレクトリで`heroku create`します

```
$ heroku create <任意のアプリ名> --buildpack heroku/python
```

`heroku create`するとHerokuのURLが発行されます
このURLをtwilioAPIコンソールのwebookのところに登録します（忘れやすいので注意）

スクリプトをherokuへ`push`します

```
$ git add -A
$ git commit -m “hogehoge”
$ git push heroku master
```

ざらざらと文字が流れます
masterに無事pushされ、エラーが出なければok

```
$ heroku open
```
すると、pushした内容が正しければ、ブラウザーでアプリが表示されるはず。

ローカルで正しく動作していたのに動かない場合は、

- requirements.txt に書き足していないモジュールは無いか
- Procfileの書き方は正しいか

を確認してみます（たいていProcfileの方に何かしら問題がある）

上記の一連の内容は[こちらのQiita記事](http://qiita.com/sqrtxx/items/2ae41d5685e07c16eda5)にすっきりまとまっていて参考になりました



## Procfileについて
Herokuにpushしたアプリケーションの実行コマンドはProcfileなるものに書いておかなければならないようです(Procfileについての詳細は[こちら](https://github.com/herokaijp/devcenter/wiki/procfile)に書かれていますが、よくわからなくてもとりあえず以下のように書けば動きます)。

gunicornなるものを使ってProcfileを書く場合、[こちらのブログ](http://shkh.hatenablog.com/entry/2013/01/01/192857)を真似して書いてみました。

例えば main.py という名前のFlaskのスクリプトをHeroku上で実行したい時、Procfileには

`web: gunicorn main:app --log-file -` という一行だけを書いて保存しておけば良いらしいです。（ `python main.py` ではない。）

（`gunicorn`はいろんな記事でこれを使うことが推奨されているっぽいので、実はよくわかっていないですがとりあえず使ってます。もしも`gunicorn`を使わない場合は、port 5000でFlaskやdjangoが立ち上がるときに問題がでるらしいです）

# Herokuにpushしたアプリを確認する

現在Herokuにpushされているアプリを確認します（無料枠では最大5つまで）

```
$ heroku apps
```

不要なアプリの削除

```
$ heroku destroy --app <アプリ名>
#-> 再度確認のためアプリ名をタイプする
```

現在 herokuのサーバーで動いているスクリプトを確認します（無料枠で起動できる残り時間なども表示される）

```
$ heroku ps
```

logの確認

```
$ heroku logs # —tailオプションでlogのtailが見れる
```

その他のHeroku CLIについてはこちらを参照

- [Heroku CLI 簡単リファレンス](http://qiita.com/histori/items/3ce08a2926c22c63626b)

# その他

## mp3ファイルのトリミング
APIによって録音されたユーザのメッセージデータは[https://jp.twilio.com/login/kddi-web](https://jp.twilio.com/login/kddi-web)からログインした先にあるコンソールからDLなどができます。（どれくらいのメッセージ数が保存できるかきちんと調べていませんが、自分の場合は60件は保存されているのを確認しました）

データの保存形式はmp3やWAVなどが選べたと思います。自分の場合はmp3で音声データをDLしていました。
音声データの先頭部分やお尻の部分を切り取って簡単に編集したい場合はこちらが無料で使え、かつ便利でした。ブラウザ上で簡単に音声トリミングができます（特に上限ファイル数なども無し）。

- [Online MP3 Cutter](http://mp3cut.net/ja/)

## Play動詞とm4aファイル問題

iPhoneのボイスレコーダーアプリが標準で出力するm4aファイルは twilioAPIのplay動詞で再生出来ないので注意！（mp3が無難）  

- 参照: [https://jp.twilio.com/docs/api/twiml/play](https://jp.twilio.com/docs/api/twiml/play)

## anacondaで仮想環境を作る
vertualenvではなくanacondaで環境を切りたい人用（通常はvertualenvを使えば良いと思う）

anacondaでenvを創ってそこに出入りする方法は以下

- 参考: [データサイエンティストを目指す人のpython環境構築 2016](http://qiita.com/y__sama/items/5b62d31cb7e6ed50f02c)

<br>
仮想環境の構築

```
$ conda create -n <環境名> python=<バージョン> <インストールしたいライブラリをスペース区切りで書く>
```

例: conda create -n py_ver2 python=2.7 numpy scipy pandas jupyter

<br>

anaconda環境をまとめて、新たな環境として作ることも可能（anacondaでanacondaの仮想環境を作るということ）

```
$ conda create -n anaconda2 python=2.7 anaconda
```


現在存在する condaで作った仮想環境一覧を表示

```
$ conda env list
```

仮想環境に入る

```
$ source activate <環境名>
```

仮想環境から抜ける

```
$ source deactivate
```


# 参考
上記のリンク以外で参考にしたページ

- [TwilioAPIで発生するユーザーリクエストのステータス・コード一覧](https://jp.twilio.com/docs/api/rest/request)
- [TwilioAPIのアラートトリガー一覧](https://jp.twilio.com/docs/api/errors/app-monitor-triggers)

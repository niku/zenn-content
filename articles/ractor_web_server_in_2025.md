---
title: "RactorベースのWebサーバーを書く(2025年版)"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Ruby, Ractor]
published: true
---

Ruby の並行処理 Ractor は 2025 年 7 月現在どのくらいよくありそうな Ruby プログラミングで書けそうになっているかを測るため、Ractor を使った簡潔な Web Server を作り、その次に Rack に対応させてみる。

後述する Ractor の新しい API、Ractor::Port を使うために、Ruby は開発版を使っている。Ruby 3.5 には入る予定のようなので Ruby 3.5 以降であれば開発版でなくても動くはずだ。

2020 年の 9 月に書かれた[Writing a Ractor-based web server](https://kirshatrov.com/posts/ruby-ractor-web-server)、その続編として 2020 年の 12 月に書かれた[Writing a Ractor-based web server: part II](https://kirshatrov.com/posts/ractor-web-server-part-two)が、整理されていて読みやすかったので、今回の記事の話の進め方もこれを真似している。

# Web サーバー

Web サーバーというのは TCP ソケットを accept し、そこから read し、HTTP ヘッダを解析し、HTTP ボディを返すような、実装が容易なテキストベースのプロトコルだ。

以下がクライアントから Web サーバーに送られてくるリクエストの例だ。実装の面から考えると、Web サーバーは、クライアントのソケットからこの文字列を read することになる。

```
GET / HTTP/1.1
Host: localhost:10000
User-Agent: curl/7.64.1
Accept: */*
```

そしてレスポンスの例だ。実装の面から考えると、クライアントのソケットにこの文字列を write することになる。

```
HTTP/1.1 200
Content-Type: text/html

Hello world
```

私たちも、AppSignal が投稿した[Building a 30 line HTTP server in Ruby](https://blog.appsignal.com/2016/11/23/ruby-magic-building-a-30-line-http-server-in-ruby.html)から始める。

```ruby
require 'socket'
server = TCPServer.new(8080)

while session = server.accept
  request = session.gets
  puts request

  session.print "HTTP/1.1 200\r\n"
  session.print "Content-Type: text/html\r\n"
  session.print "\r\n"
  session.print "Hello world! The time is #{Time.now}"

  session.close
end
```

実際に irb と curl で試してもうまく動く。

# Ractor を導入する

Ractor を始めるなら Ruby のリポジトリにある [ドキュメント](https://github.com/ruby/ruby/blob/8f54b5bb93a6dc703f0450479f215a9e2584a190/doc/ractor.md)を読んで概要を掴んでからがおすすめだ。
ただこのドキュメントも 2025 年 7 月 14 日現在では最新化されていないところがある。最近 Ractor::Port という機能が導入された。こちらは [`Ractor::Port` ― Ractor の API を一新した話](https://product.st.inc/entry/2025/06/24/110606) に詳しい。今回は積極的に一新された API の方を使っていく。

さてそれでは先程の例を Ractor と組み合わせていこう。

```ruby
require 'etc'
require 'socket'
server = TCPServer.new(8080)
CPU_COUNT = Etc.nprocessors
workers = CPU_COUNT.times.map do
  Ractor.new do
    loop do
      # receive TCPSocket
      s = Ractor.receive

      request = s.gets
      puts request

      s.print "HTTP/1.1 200\r\n"
      s.print "Content-Type: text/html\r\n"
      s.print "\r\n"
      s.print "Hello world! The time is #{Time.now}\n"
      s.close
    end
  end
end

loop do
  conn, _ = server.accept
  # pass TCPSocket to one of the workers
  workers.sample.send(conn, move: true)
end
```

このコードでは最初に CPU の数と同数のワーカーを動かし、メインスレッドでソケットの接続を listen、accept したコネクションをランダムな Ractor に送っている。このコードを実行して、curl でアクセスすると期待通りに動いていることが確かめられる。

# 手があいているワーカーがリクエストをとる

`worker.sample` を使う方式は動作するものの、効率的とは言えない。ランダムに割り当てられたワーカーがまだ過去のリクエストを処理している最中かもしれないためだ。そこで、手があいているワーカーがリクエストをとる形に変更する。

```ruby
require 'etc'
require 'socket'
controller = Ractor::Port.new
server = TCPServer.new(8080)
CPU_COUNT = Etc.nprocessors
workers = CPU_COUNT.times.map do
  Ractor.new(controller) do |controller|
    loop do
      controller.send(Ractor.current)
      s = Ractor.receive

      request = s.gets
      puts request

      s.print "HTTP/1.1 200\r\n"
      s.print "Content-Type: text/html\r\n"
      s.print "\r\n"
      s.print "Hello world! The time is #{Time.now}\n"
      s.close
    end
  end
end

loop do
  conn, _ = server.accept
  worker = controller.receive
  worker.send(conn, move: true)
end
```

:::details 元記事との違い

https://kirshatrov.com/posts/ruby-ractor-web-server の記事は以下のコードで共有キューを表現していた。（pipe を宣言しているところが Queue として動作している）
Ractor::Port を導入した際に `Ractor#take` は廃止となったので、今回は Ractor::Port での表現に変えている。

```ruby
require 'socket'

# pipe aka a queue
pipe = Ractor.new do
  loop do
    Ractor.yield(Ractor.recv, move: true)
  end
end

CPU_COUNT = 4
workers = CPU_COUNT.times.map do
  Ractor.new(pipe) do |pipe|
    loop do
      s = pipe.take

      data = s.recv(1024)
      puts data.inspect

      s.print "HTTP/1.1 200\r\n"
      s.print "Content-Type: text/html\r\n"
      s.print "\r\n"
      s.print "Hello world!\n"
      s.close
    end
  end
end

server = TCPServer.new(8080)
loop do
  conn, _ = server.accept
  pipe.send(conn, move: true)
end
```

:::

上のコードでの `controller.receive` で取得する worker は、処理をすぐに開始できる。`controller.send(Ractor.current)` で controller に登録した後にだけその worker は呼び出されることになるが、その時点で worker は必ずリクエストを処理できる状態（リクエスト処理完了後か、初期化済の状態）になっているためである。

こうして処理中のワーカーにさらに処理を送ることがなくなり、負荷分散できるようになった。

# 空いているワーカーがいなくても accept をすぐにすませる

先ほどのコードの最後

```ruby
loop do
  conn, _ = server.accept
  worker = controller.receive
  worker.send(conn, move: true)
end
```

に注目しよう。このコードでは、空いているワーカーがでてくるまで `controller.receive` のところで処理が止まる（ブロッキングする）。処理が止まっているので、サーバーに空いているワーカーがいない状態で新しいクライアントが接続してきても `server.accept` は行われない。

効率のためには `server.accept` は常に実施したい。そこで [Puma の Architecture](https://github.com/puma/puma/blob/v6.6.0/docs/architecture.md) に似せて、常に `accept` を続ける別の Ractor （Puma で言うところの Reactor クラス）を立ち上げて、ソケットの準備が整うのを待ってから処理担当のワーカーに渡す方式へと改善する。

このプログラムは全ての処理がメインの Ractor 以外で行われるようになっている。そのため最後の `Ractor.select(dispatcher, *workers, listener)` を書かないと、処理が終わっていないのにプログラムが終わってしまう。また副次的な効果として `Ractor.select(dispatcher, *workers, listener)` を含むループで、dispatcher, worker, listener それぞれの処理が意図せずに終わってしまった場合の対処を書けるようになった。（このコードではコメントにとどめている）

```ruby
require 'etc'
require 'socket'

port = Ractor::Port.new
dispatcher = Ractor.new(port) do |port|
  controller = Ractor::Port.new
  todo = Ractor::Port.new
  port.send([controller, todo])
  loop do
    worker = controller.receive
    worker.send(todo.receive, move: true)
  end
end
controller, todo = port.receive
port.close

CPU_COUNT = Etc.nprocessors
workers = CPU_COUNT.times.map do
  Ractor.new(controller) do |controller|
    loop do
      controller.send(Ractor.current)
      s = Ractor.receive

      request = s.gets
      puts request

      s.print "HTTP/1.1 200\r\n"
      s.print "Content-Type: text/html\r\n"
      s.print "\r\n"
      s.print "Hello world! The time is #{Time.now}\n"
      s.close
    end
  end
end

listener = Ractor.new(todo) do |todo|
  server = TCPServer.new(8080)
  loop do
    conn, _ = server.accept
    todo.send(conn, move: true)
  end
end

loop do
  Ractor.select(dispatcher, *workers, listener)
  # if the line above returned, one of the dispatcher, the workers or the listener has crashed
end
```

# リクエストヘッダを解析する

Web サーバー実装の次のステップとして、リクエストヘッダーを読みこんで解析する HTTP パーサーを組み込む。手近な HTTP パーサーとして、Ruby で Web サーバーを実装した [WEBrick](https://rubygems.org/gems/webrick) がある。この gem の一部を利用する。

ただ、そのままでは使えず、簡単な以下のコードでもエラーになる。

```ruby
require 'webrick'
Ractor.shareable?(WEBrick::Config::HTTP) # => false
Ractor.new {  WEBrick::HTTPRequest.new(WEBrick::Config::HTTP) }
# can not access non-shareable objects in constant WEBrick::Config::HTTP by non-main Ractor. (Ractor::IsolationError)
```

これは `WEBrick::Config::HTTP` が

- main の Ractor で定義されている、つまりある Ractor 上で定義されたものを別の Ractor 上で使おうとしている
- 設定のためのオブジェクトをいくつか持つ、可変する（mutable な）ハッシュオブジェクトなので、[sharable](https://docs.ruby-lang.org/en/3.4/Ractor.html#method-c-shareable-3F) ではない

の 2 つの条件に合致するためだ。

定数を同じ Ractor 上で定義するか、オブジェクトを不変（immutable）へと変更することで sharable にするとこの問題は解決する。この定数はライブラリの中で定義されているので、どの Ractor で定義するかを操作するのは難しい。sharable になるようにオブジェクトを操作することで解決を目指す。Ractor にはそれをするのに便利な [`Ractor.make_shareable`](https://docs.ruby-lang.org/en/3.4/Ractor.html#method-c-make_shareable) という関数がある。これを使うと、以下のようにエラーが解消される。

```ruby
require 'webrick'
Ractor.shareable?(WEBrick::Config::HTTP) # => false
Ractor.make_shareable(WEBrick::Config::HTTP)
Ractor.shareable?(WEBrick::Config::HTTP) # => true
Ractor.new {  WEBrick::HTTPRequest.new(WEBrick::Config::HTTP) }
```

同様に、処理を進めると `WEBrick::HTTPUtils::HEADER_CLASSES` も同じエラーになるので、shareable なオブジェクトへと強制変更しておく。

```ruby
require 'etc'
require 'socket'
require 'webrick'

Ractor.make_shareable(WEBrick::Config::HTTP)
Ractor.make_shareable(WEBrick::HTTPUtils::HEADER_CLASSES)

port = Ractor::Port.new
dispatcher = Ractor.new(port) do |port|
  controller = Ractor::Port.new
  todo = Ractor::Port.new
  port.send([controller, todo])
  loop do
    worker = controller.receive
    worker.send(todo.receive, move: true)
  end
end
controller, todo = port.receive
port.close

CPU_COUNT = Etc.nprocessors
workers = CPU_COUNT.times.map do
  Ractor.new(controller) do |controller|
    loop do
      controller.send(Ractor.current)
      s = Ractor.receive

      request = WEBrick::HTTPRequest.new(WEBrick::Config::HTTP.merge(RequestTimeout: nil))
      request.parse(s)
      puts request.inspect

      s.print "HTTP/1.1 200\r\n"
      s.print "Content-Type: text/html\r\n"
      s.print "\r\n"
      s.print "Hello world! The time is #{Time.now}\n"
      s.close
    end
  end
end

listener = Ractor.new(todo) do |todo|
  server = TCPServer.new(8080)
  loop do
    conn, _ = server.accept
    todo.send(conn, move: true)
  end
end

loop do
  Ractor.select(dispatcher, *workers, listener)
  # if the line above returned, one of the dispatcher, the workers or the listener has crashed
end
```

# Rack アプリケーションを動かす

Ruby で書かれた実用的な Web アプリケーションは、殆どの場合 Web サーバーとのインターフェースとして Rack を利用している。
今作っている Ractor を使った Web サーバーも、Rack と互換性を持つものにしてみよう。

この Web サーバーと Rack インターフェースを取り持つ部分は[第 3 章: 自作の Rack サーバを実装してみよう](https://github.com/hogelog/kaigionrails-2024-rack-workshop/blob/main/03-server.md)でわかりやすく解説されている。同じように Rack インターフェースに適合するように Web サーバーの Handler というものを作って登録する。

既に実装されている WEBrick の場合では [`Rack::Handler::WeBrick#service`](https://github.com/rack/rackup/blob/v2.2.1/lib/rackup/handler/webrick.rb#L91) メソッドで主要な処理を行っている。

1. `WEBrick::HTTPRequest` を入力として受け取り、[Rack env に変換する](hhttps://github.com/rack/rackup/blob/v2.2.1/lib/rackup/handler/webrick.rb#L92-L109)
2. その env を使って [Rack app を call する](https://github.com/rack/rackup/blob/v2.2.1/lib/rackup/handler/webrick.rb#L111)
3. Rack app の結果を [`WEBrick::HTTPResponse` に詰めて](https://github.com/rack/rackup/blob/v2.2.1/lib/rackup/handler/webrick.rb#L113-L153)出力する

この 2 つの資料を参考に、簡潔な Rack アプリケーションを動かせるような Handler を作る。

```ruby
require 'etc'
require 'socket'
require 'webrick'

Ractor.make_shareable(WEBrick::Config::HTTP)
Ractor.make_shareable(WEBrick::HTTPUtils::HEADER_CLASSES)
Ractor.make_shareable(WEBrick::HTTPStatus::StatusMessage)

class App
  def call(env)
    if env["PATH_INFO"] == "/"
      [200, {}, ["It works!"]]
    else
      [404, {}, ["Not Found"]]
    end
  end
end

class RactorServer
  CPU_COUNT = Etc.nprocessors

  def self.run(app, **options)
    new(app, options).start
  end

  def initialize(app, options)
    @app = app
    @options = options
  end

  def start
    port = Ractor::Port.new
    dispatcher = Ractor.new(port) do |port|
      controller = Ractor::Port.new
      todo = Ractor::Port.new
      port.send([controller, todo])
      loop do
        worker = controller.receive
        worker.send(todo.receive, move: true)
      end
    end
    controller, todo = port.receive
    port.close

    workers = CPU_COUNT.times.map do
      Ractor.new(@app, @options, controller) do |app, options, controller|
        loop do
          controller.send(Ractor.current)
          s = Ractor.receive

          request = WEBrick::HTTPRequest.new(WEBrick::Config::HTTP.merge(RequestTimeout: nil))
          request.parse(s)

          env = request.meta_vars
          status, headers, body = app.call(env)

          response = WEBrick::HTTPResponse.new(WEBrick::Config::HTTP)
          response.status = status
          body.each { |part| response.body << part }
          response.send_response(s)
        end
      end
    end

    listener = Ractor.new(todo) do |todo|
      server = TCPServer.new(8080)
      loop do
        conn, _ = server.accept
        todo.send(conn, move: true)
      end
    end

    loop do
      Ractor.select(dispatcher, *workers, listener)
      # if the line above returned, one of the dispatcher, the workers or the listener has crashed
    end
  end
end

Rackup::Handler.register :ractor_server, RactorServer

run App.new
```

このファイルを config.ru という名前で保存して rack アプリケーションとして動かす。rack アプリケーションは rackup コマンドで起動する。
起動オプションに何もつけないと、lint が含まれるデフォルトのスタックで起動するのだが、今回は簡易的な実装で Rack アプリケーションが必要とするパラメータを網羅しておらず lint でエラーになるため `--env none` で外している。

コードは https://gist.github.com/niku/44878506db183eeb82d7e14da4f558c7 にあるので、手元でも以下のようにすると動作を試せる。

```shell
gem install rackup
git clone https://gist.github.com/niku/44878506db183eeb82d7e14da4f558c7 ractor_web_server
cd ractor_web_server
rackup --server ractor_server --env none config.ru
```

![](/images/ractor_web_server_2025.png)

# まとめ

「Ruby の並行処理 Ractor は 2025 年 7 月現在どのくらいよくありそうな Ruby プログラミングで書けそうになっているか」というのを知るために試した。

Writing a Ractor-based web server が書かれた 2020 年から、Ractor 本体の改善や周囲のライブラリの Ractor 対応によって、簡易的な Web サーバーであればよくありそうな Ruby プログラミングとして、すごく特別なワークアラウンドなしでも書けることがわかった。ただ、まだいくつかの部分で `Ractor.make_shareable` が必要だった。このあたりはライブラリ側に地道に PR を送っていくことになるだろう。とはいえ `Ractor.make_shareable` のおかげで手元での対処は容易だった。

新しい `Ractor::Port` の API は私には理解しやすかった。
私の場合は Ractor の考えの元になっているアクターモデルの考え方に Erlang や Elixir を通じて親しんでいたので、ギャップが少なかったのかもしれない。
それでも、Ractor::Port を紹介するブログにも、欠点として「典型的な producer-consumer 型では、Port を土台にもう一段の抽象化が必要になる」と書かれていた通り

```ruby
port = Ractor::Port.new
dispatcher = Ractor.new(port) do |port|
  controller = Ractor::Port.new
  todo = Ractor::Port.new
  port.send([controller, todo])
  loop do
    worker = controller.receive
    worker.send(todo.receive, move: true)
  end
end
controller, todo = port.receive
port.close
```

のようなコードを思いつくのには少し時間がかかったので、便利なラッパーがあってもいいのかもしれない。（一回思いついたら平気になるような、馴れの問題かもしれない）

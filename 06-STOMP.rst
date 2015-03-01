STOMPを使ってみる
********************************************************************************

前章でMessageListenerで非同期に画像処理を行いましたが、処理結果がクライアントには返ってきていません。
今度は処理結果もメッセージングで送るようにしましょう。ここではメッセージングプロトコルとして\ `STOMP <http://stomp.github.io/>`_\ を使用します。

STOMPは「Simple (or Streaming) Text Orientated Messaging Protocol」の略で、軽量なメッセージングプロトコルです。
TCPやWebSocket上で利用できます。

本章ではSTOMP over WebSocketの簡単な使い方を学びましょう。

まずはpom.xmlに以下の依存関係を追加してください。

.. code-block:: xml

    <!-- WebSocketプログラミングに必要な諸々 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>

\ ``App``\ クラスに以下を追加します。

.. code-block:: java
    :emphasize-lines: 6-21,24-29

    @SpringBootApplication
    @RestController
    public class App {
        // ...

        @Configuration
        @EnableWebSocketMessageBroker // WebSocketに関する設定クラス
        static class StompConfig extends AbstractWebSocketMessageBrokerConfigurer {

            @Override
            public void registerStompEndpoints(StompEndpointRegistry registry) {
                registry.addEndpoint("endpoint"); // WebSocketのエンドポイント
            }

            @Override
            public void configureMessageBroker(MessageBrokerRegistry registry) {
                registry.setApplicationDestinationPrefixes("/app"); // Controllerに処理させる宛先のPrefix
                registry.enableSimpleBroker("/topic"); // queueまたはtopicを有効にする(両方可)。queueは1対1(P2P)、topicは1対多(Pub-Sub)
            }
        }

        // ...

        @MessageMapping(value = "/greet" /* 宛先名 */) // Controller内の@MessageMappingアノテーションをつけたメソッドが、メッセージを受け付ける
        @SendTo(value = "/topic/greetings") // 処理結果の送り先
        String greet(String name) {
            log.info("received {}", name);
            return "Hello " + name;
        }

        // ※ handleHelloMessageは削除(またはコメントアウト)しておいてください。
        // ...
    }

ソースだけでは分かりにくいと思いますが、メッセージのフローは\ `下図 <http://rstoyanchev.github.io/webinar-websocket-spring/#26>`_\ のようになります。

.. figure:: ./images/stomp-message-flow.png
    :width: 60%

宛先が/topicや/queueで始まるものはメッセージブローカー(仲介役)が直接ハンドリングします。
宛先が/appから始まるものはControllerに渡って処理され、その処理結果がメッセージブローカーに渡ります。

メッセージブローカーによって制御されたメッセージは、その宛先を購読しているクライアントへと送られます。


次に\ `Stomp.js <http://jmesnil.net/stomp-websocket/doc/>`_\ を使ってクライアントを作りましょう。\ :file:`src/main/resources/static`\ に\ :file:`hello.html`\ を作成してください。

.. note::

    Spring Bootでは\ :file:`src/main/resources/static`\ 以下が静的リソース置き場になります。このディレクトリにファイルを置くと、コンテキストパスから相対的にアクセスできます。

.. code-block:: html

    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <title>Hello STOMP</title>
    </head>
    <body>
    <div>
        <button id="connect">Connect</button>
        <button id="disconnect" disabled="disabled">Disconnect</button>
    </div>
    <div>
        <input type="text" id="name" placeholder="Your Name">
        <button id="send" disabled="disabled">Send</button>
        <div id="response"></div>
    </div>
    </body>
    <script src="stomp.js"></script>
    <script type="text/javascript">
        var HelloStomp = function () {
            this.connectButton = document.getElementById('connect');
            this.disconnectButton = document.getElementById('disconnect');
            this.sendButton = document.getElementById('send');

            this.connectButton.addEventListener('click', this.connect.bind(this));
            this.disconnectButton.addEventListener('click', this.disconnect.bind(this));
            this.sendButton.addEventListener('click', this.sendName.bind(this));
        };
        HelloStomp.prototype.connect = function () {
            var socket = new WebSocket('ws://localhost:8080/endpoint'); // エンドポイントのURL
            this.stompClient = Stomp.over(socket); // WebSocketを使ったStompクライアントを作成
            this.stompClient.connect({}, function (frame) {
                this.connected();
                console.log('Connected: ' + frame);
                this.stompClient.subscribe('/topic/greetings', function (message) { // 宛先が'/topic/greetings'のメッセージを購読
                    this.showGreeting(message.body); // コールバック処理
                }.bind(this));
            }.bind(this));
        };
        HelloStomp.prototype.connected = function () {
            this.connectButton.disabled = true;
            this.disconnectButton.disabled = false;
            this.sendButton.disabled = false;
        };
        HelloStomp.prototype.disconnect = function () {
            this.connectButton.disabled = false;
            this.disconnectButton.disabled = true;
            this.sendButton.disabled = true;
        };
        HelloStomp.prototype.sendName = function () {
            var name = document.getElementById('name').value;
            this.stompClient.send('/app/greet', {/* headers */}, name); // 宛先'/app/greet'へメッセージを送信
        };
        HelloStomp.prototype.showGreeting = function (message) {
            var response = document.getElementById('response');
            var p = document.createElement('p');
            p.appendChild(document.createTextNode(message));
            response.insertBefore(p, response.children[0]);
        };

        new HelloStomp();
    </script>
    </html>

Stomp.jsを\ :file:`src/main/resources/static`\ にダウンロードしましょう。

.. code-block:: console

    $ cd src/main/resources/static
    $ wget https://raw.github.com/jmesnil/stomp-websocket/master/lib/stomp.js

\ ``App``\ クラスを起動し、http://localhost:8080/hello.html\ にアクセスしてください。


.. figure:: ./images/hello-html-01.png
    :width: 60%

「Connect」ボタンを押して、フォームに名前を入力し、「Send」ボタンを押してください。

.. figure:: ./images/hello-html-02.png
    :width: 60%

結果が返ってきました。今回は宛先をTopicにしているため、他のタブで別途Connectすれば全てのタブに結果が表示されます。

.. note::

    \ ``mvn spring-boot:run``\ で\ ``App``\ クラスを起動すれば、静的リソースの変更が即反映されるので開発中は便利です。



STOMPの簡単な使い方を学びました。以上で本章は終了です。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc06」をつけてツイートしてください。

次章ではJMSのMessageListenerの処理結果をSTOMPの宛先に送り、クライアントで表示させましょう。
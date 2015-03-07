WebRTCで撮った写真を顔変換サービスに送信
********************************************************************************

これまで学んできたことを統合しましょう。

まずは\ :file:`face.html`\ に\ :file:`camera.html`\ の内容を追加します。カメラのスナップショットを撮ったら、
その画像を宛先\ ``app/faceConverter``\ へ送ります(サーバーサイドの処理は後ほど追加します)。

.. code-block:: html
    :emphasize-lines: 13-16,23-25,33-38,43-55,125,128-147

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
        <video autoplay width="400" height="300"></video>
        <img id="snapshot" src="" width="400" height="300">
        <canvas style="display:none;" width="400" height="300"></canvas>
        <br>

        <div id="response"></div>
    </div>
    </body>
    <script src="stomp.js"></script>
    <script type="text/javascript">
        // camera
        navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || window.navigator.mozGetUserMedia || navigator.msGetUserMedia;
        window.URL = window.URL || window.webkitURL;

        /**
         * 初期化処理
         */
        var HelloStomp = function () {
            this.connectButton = document.getElementById('connect');
            this.disconnectButton = document.getElementById('disconnect');
            // this.sendButton = document.getElementById('send'); この行は削除
            this.video = document.querySelector('video');
            this.canvas = document.querySelector('canvas');
            this.canvasContext = this.canvas.getContext('2d');
            this.snapshot = document.getElementById('snapshot');
            this.canvasContext.globalAlpha = 1.0;

            // イベントハンドラの登録
            this.connectButton.addEventListener('click', this.connect.bind(this));
            this.disconnectButton.addEventListener('click', this.disconnect.bind(this));
            // this.sendButton.addEventListener('click', this.sendName.bind(this)); この行は削除
            this.video.addEventListener('click', this.takeSnapshot.bind(this));

            // getUserMedia API
            navigator.getUserMedia({video: true, audio: false},
                    function (stream) {
                        this.video.src = window.URL.createObjectURL(stream);
                        this.localMediaStream = stream;
                    }.bind(this),
                    function (error) {
                        alert(JSON.stringify(error));
                    }
            );
        };

        /**
         * エンドポイントへの接続処理
         */
        HelloStomp.prototype.connect = function () {
            var socket = new WebSocket('ws://' + location.host + '/endpoint'); // エンドポイントのURL
            this.stompClient = Stomp.over(socket); // WebSocketを使ったStompクライアントを作成
            this.stompClient.debug = null; // デバッグログを出さない
            this.stompClient.connect({}, this.onConnected.bind(this)); // エンドポイントに接続し、接続した際のコールバックを登録
        };

        /**
         * エンドポイントへ接続したときの処理
         */
        HelloStomp.prototype.onConnected = function (frame) {
            console.log('Connected: ' + frame);
            // 宛先が'/topic/greetings'のメッセージを購読し、コールバック処理を登録
            this.stompClient.subscribe('/topic/greetings', this.onSubscribeGreeting.bind(this));
            // 宛先が'/topic/faces'のメッセージを購読し、コールバック処理を登録
            this.stompClient.subscribe('/topic/faces', this.onSubscribeFace.bind(this));
            this.setConnected(true);
        };

        /**
         * 宛先'/topic/greetings'なメッセージを受信したときの処理
         */
        HelloStomp.prototype.onSubscribeGreeting = function (message) {
            var response = document.getElementById('response');
            var p = document.createElement('p');
            p.appendChild(document.createTextNode(message.body));
            response.insertBefore(p, response.children[0]);
        };

        /**
         * 宛先'/topic/faces'なメッセージを受信したときの処理
         */
        HelloStomp.prototype.onSubscribeFace = function (message) {
            var response = document.getElementById('response');
            var img = document.createElement('img');
            img.setAttribute("src", "data:image/png;base64," + message.body); // Base64エンコードされた画像をそのまま表示する
            response.insertBefore(img, response.children[0]);
        };

        /**
         * 宛先'/app/greet'へのメッセージ送信処理
         */
        HelloStomp.prototype.sendName = function () {
            var name = document.getElementById('name').value;
            this.stompClient.send('/app/greet', {}, name); // 宛先'/app/greet'へメッセージを送信
        };

        /**
         * 接続切断処理
         */
        HelloStomp.prototype.disconnect = function () {
            if (this.stompClient) {
                this.stompClient.disconnect();
                this.stompClient = null;
            }
            this.setConnected(false);
        };

        /**
         * ボタン表示の切り替え
         */
        HelloStomp.prototype.setConnected = function (connected) {
            this.connectButton.disabled = connected;
            this.disconnectButton.disabled = !connected;
            // this.sendButton.disabled = !connected; この行は削除
        };

        /**
         * カメラのスナップショットを取得
         */
        HelloStomp.prototype.takeSnapshot = function () {
            this.canvasContext.drawImage(this.video, 0, 0, 400, 300);
            var dataUrl = this.canvas.toDataURL('image/jpeg');
            this.snapshot.src = dataUrl;
            this.sendFace(dataUrl);
        };

        /**
         * 顔画像の送信
         */
        HelloStomp.prototype.sendFace = function (dataUrl) {
            if (this.stompClient) {
                this.stompClient.send("/app/faceConverter", {}, dataUrl.replace(/^.*,/, '')); // 宛先'/app/faceConverter'へメッセージを送信
            } else {
                alert('not connected!');
            }
        };

        new HelloStomp();
    </script>
    </html>

サーバーサイドに、宛先\ ``app/faceConverter``\ に対する処理を追加しましょう。

Base64でエンコードされた画像を\ ``byte[]``\ にデコードして、JMSのMessageListenerへ送信するだけです。

.. code-block:: java
    :emphasize-lines: 6-10

    @SpringBootApplication
    @RestController
    public class App {
        // ...

        @MessageMapping(value = "/faceConverter")
        void faceConverter(String base64Image) {
            Message<byte[]> message = MessageBuilder.withPayload(Base64.getDecoder().decode(base64Image)).build();
            jmsMessagingTemplate.send("faceConverter", message);
        }

        // ...
    }

\ ``App``\ クラスを再起動し、http://localhost:8080/face.html\ にアクセスしてください。

「Connect」ボタンを押して接続したら、カメラの画像をクリックしてください。スナップショットが保存され、サーバーへ送信されます。
しばらくすると処理結果を受信し、表示します。連続してクリックしてもスムーズに処理されることを確認してください。

最後に、カメラ画像だけでなく、ローカルファイルも送信できるように機能追加しましょう。\ :file:`face.html`\ を以下のように修正してください。

.. code-block:: html
    :emphasize-lines: 17,34,45,126,150-164

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
        <video autoplay width="400" height="300"></video>
        <img id="snapshot" src="" width="400" height="300">
        <canvas style="display:none;" width="400" height="300"></canvas>
        <br>
        <input id="files" type="file" disabled="disabled" multiple>

        <div id="response"></div>
    </div>
    </body>
    <script src="stomp.js"></script>
    <script type="text/javascript">
        // camera
        navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || window.navigator.mozGetUserMedia || navigator.msGetUserMedia;
        window.URL = window.URL || window.webkitURL;

        /**
         * 初期化処理
         */
        var HelloStomp = function () {
            this.connectButton = document.getElementById('connect');
            this.disconnectButton = document.getElementById('disconnect');
            this.files = document.getElementById('files');
            this.video = document.querySelector('video');
            this.canvas = document.querySelector('canvas');
            this.canvasContext = this.canvas.getContext('2d');
            this.snapshot = document.getElementById('snapshot');
            this.canvasContext.globalAlpha = 1.0;

            // イベントハンドラの登録
            this.connectButton.addEventListener('click', this.connect.bind(this));
            this.disconnectButton.addEventListener('click', this.disconnect.bind(this));
            this.video.addEventListener('click', this.takeSnapshot.bind(this));
            this.files.addEventListener('change', this.sendFiles.bind(this));

            // getUserMedia API
            navigator.getUserMedia({video: true, audio: false},
                    function (stream) {
                        this.video.src = window.URL.createObjectURL(stream);
                        this.localMediaStream = stream;
                    }.bind(this),
                    function (error) {
                        alert(JSON.stringify(error));
                    }
            );
        };

        /**
         * エンドポイントへの接続処理
         */
        HelloStomp.prototype.connect = function () {
            var socket = new WebSocket('ws://' + location.host + '/endpoint'); // エンドポイントのURL
            this.stompClient = Stomp.over(socket); // WebSocketを使ったStompクライアントを作成
            this.stompClient.debug = null; // デバッグログを出さない
            this.stompClient.connect({}, this.onConnected.bind(this)); // エンドポイントに接続し、接続した際のコールバックを登録
        };

        /**
         * エンドポイントへ接続したときの処理
         */
        HelloStomp.prototype.onConnected = function (frame) {
            console.log('Connected: ' + frame);
            // 宛先が'/topic/greetings'のメッセージを購読し、コールバック処理を登録
            this.stompClient.subscribe('/topic/greetings', this.onSubscribeGreeting.bind(this));
            // 宛先が'/topic/faces'のメッセージを購読し、コールバック処理を登録
            this.stompClient.subscribe('/topic/faces', this.onSubscribeFace.bind(this));
            this.setConnected(true);
        };

        /**
         * 宛先'/topic/greetings'なメッセージを受信したときの処理
         */
        HelloStomp.prototype.onSubscribeGreeting = function (message) {
            var response = document.getElementById('response');
            var p = document.createElement('p');
            p.appendChild(document.createTextNode(message.body));
            response.insertBefore(p, response.children[0]);
        };

        /**
         * 宛先'/topic/faces'なメッセージを受信したときの処理
         */
        HelloStomp.prototype.onSubscribeFace = function (message) {
            var response = document.getElementById('response');
            var img = document.createElement('img');
            img.setAttribute("src", "data:image/png;base64," + message.body); // Base64エンコードされた画像をそのまま表示する
            response.insertBefore(img, response.children[0]);
        };

        /**
         * 宛先'/app/greet'へのメッセージ送信処理
         */
        HelloStomp.prototype.sendName = function () {
            var name = document.getElementById('name').value;
            this.stompClient.send('/app/greet', {}, name); // 宛先'/app/greet'へメッセージを送信
        };

        /**
         * 接続切断処理
         */
        HelloStomp.prototype.disconnect = function () {
            if (this.stompClient) {
                this.stompClient.disconnect();
                this.stompClient = null;
            }
            this.setConnected(false);
        };

        /**
         * ボタン表示の切り替え
         */
        HelloStomp.prototype.setConnected = function (connected) {
            this.connectButton.disabled = connected;
            this.disconnectButton.disabled = !connected;
            this.files.disabled = !connected;
        };

        /**
         * カメラのスナップショットを取得
         */
        HelloStomp.prototype.takeSnapshot = function () {
            this.canvasContext.drawImage(this.video, 0, 0, 400, 300);
            var dataUrl = this.canvas.toDataURL('image/jpeg');
            this.snapshot.src = dataUrl;
            this.sendFace(dataUrl);
        };

        /**
         * 顔画像の送信
         */
        HelloStomp.prototype.sendFace = function (dataUrl) {
            if (this.stompClient) {
                this.stompClient.send("/app/faceConverter", {}, dataUrl.replace(/^.*,/, ''));
            } else {
                alert('not connected!');
            }
        };

        /**
         * 選択した画像ファイルを送信
         */
        HelloStomp.prototype.sendFiles = function (event) {
            var input = event.target;
            for (var i = 0; i < input.files.length; i++) {
                var file = input.files[i];
                var reader = new FileReader();
                reader.onload = function (event) {
                    var dataUrl = event.target.result;
                    this.sendFace(dataUrl);
                }.bind(this);
                reader.readAsDataURL(file);
            }
        };

        new HelloStomp();
    </script>
    </html>

再度、http://localhost:8080/face.html\ にアクセスしてファイルアップロードしてみてください。複数ファイルを一度に送信できます。

以上で本章は終了です。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc09」をつけてツイートしてください。


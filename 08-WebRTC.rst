WebRTCを使ってみる
********************************************************************************

本章ではまずはWebRTCのgetUserMedia APIカメラにアクセスしてみましょう。

WebRTC(Web Real-Time Communication)ではリアルタイムコミュニケーション用のAPIが用意されていますが、本ハンズオンではgetUserMedia APIのみ使用します。

\ :file:`src/main/resources/static/camera.html`\ を作成して、以下の内容を記述してください。


.. code-block:: html

    <!doctype html>
    <html>
    <head>
        <title>Cameraテスト</title>
    </head>
    <body>
    <video autoplay width="400" height="300"></video>
    <img src="" width="400" height="300">
    <canvas style="display:none;" width="400" height="300"></canvas>

    <script type="text/javascript">
        navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || window.navigator.mozGetUserMedia || navigator.msGetUserMedia;
        window.URL = window.URL || window.webkitURL;

        var video = document.querySelector('video');
        var canvas = document.querySelector('canvas');
        var ctx = canvas.getContext('2d');
        var localMediaStream;

        navigator.getUserMedia({video: true, audio: false},
                function (stream) {
                    video.src = window.URL.createObjectURL(stream);
                    localMediaStream = stream;
                },
                function (error) {
                    alert(JSON.stringify(error));
                }
        );

        function takeSnapshot() {
            if (localMediaStream) {
                ctx.drawImage(video, 0, 0, 400, 300);
                document.querySelector('img').src = canvas.toDataURL('image/webp');
            }
        }
        video.addEventListener('click', takeSnapshot, false);
    </script>
    </body>
    </html>

http://localhost:8080/camera.html\ にアクセスしてください。

.. figure:: ./images/camera-html-01.png
    :width: 80%

カメラアクセスへの許可を確認されますので、「許可」をクリックしてください。そうするとカメラの結果が左側に表示されます。

左のカメラ画像をクリックすると、右側にスナップショットして表示されます。

.. figure:: ./images/camera-html-02.png
    :width: 80%

以上で本章は終了です。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc08」をつけてツイートしてください。

次章ではいよいよカメラ画像をサーバーに送信し、撮った画像が変換されて表示するようにします。
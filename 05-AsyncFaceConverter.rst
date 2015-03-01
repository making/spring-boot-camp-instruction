JMSで画像変換を非同期処理
********************************************************************************

前章で学んだJMSを使って、画像データをメッセージに詰めて送信し、画像変換処理をMessageListenerで非同期処理するようにしましょう。

まずは画像処理用のキューを追加するために\ :file:`application.yml`\ を以下のように修正します。

.. code-block:: yaml
    :emphasize-lines: 8

    spring:
      thymeleaf.cache: false
      main.show-banner: false
      hornetq:
        mode: embedded
        embedded:
          enabled: true
          queues: hello,faceConverter # 宛先名

次に送信処理を作成しましょう。

\ ``JmsMessagingTemplate``\ はデフォルトでは

* \ ``String``\
* \ ``byte[]``\
* \ ``Map``\
* \ ``Serializable``\

の変換に対応しています。今回は非効率的ですが、送られた\ ``javax.servlet.http.Part``\ から取得した画像データを\ ``byte``\ 配列に変換し、
MessageListener側で\ ``byte``\ 配列から\ ``BufferedImage``\ に変換し、OpenCVに渡します。

まずはControllerにリクエスト受付処理を追加しましょう。

.. code-block:: java
    :emphasize-lines: 4,12-18

    package kanjava;

    // ...
    import org.springframework.util.StreamUtils;
    // ...

    @SpringBootApplication
    @RestController
    public class App {
        // ...

        @RequestMapping(value = "/queue", method = RequestMethod.POST)
        String queue(@RequestParam Part file) throws IOException {
            byte[] src = StreamUtils.copyToByteArray(file.getInputStream()); // InputStream -> byte[]
            Message<byte[]> message = MessageBuilder.withPayload(src).build(); // byte[]を持つMessageを作成
            jmsMessagingTemplate.send("faceConverter", message); // convertAndSend("faceConverter", src)でも可
            return "OK";
        }

        // ...
    }

次に、宛先\ ``faceConverter``\ に対するMessageListenerを追加しましょう。


.. code-block:: java
    :emphasize-lines: 16-25

    @SpringBootApplication
    @RestController
    public class App {
        // ...

        @RequestMapping(value = "/queue", method = RequestMethod.POST)
        String queue(@RequestParam Part file) throws IOException {
            byte[] src = StreamUtils.copyToByteArray(file.getInputStream());
            Message<byte[]> message = MessageBuilder.withPayload(src).build();
            jmsMessagingTemplate.send("faceConverter", message);
            return "OK";
        }

        // ...

        @JmsListener(destination = "faceConverter", concurrency = "1-5")
        void convertFace(Message<byte[]> message) throws IOException {
            log.info("received! {}", message);
            try (InputStream stream = new ByteArrayInputStream(message.getPayload())) { // byte[] -> InputStream
                Mat source = Mat.createFrom(ImageIO.read(stream)); // InputStream -> BufferedImage -> Mat
                faceDetector.detectFaces(source, FaceTranslator::duker);
                BufferedImage image = source.getBufferedImage();
                // do nothing...
            }
        }
    }

ここまでの内容を組み合わせれば、内容を理解できると思います。

.. code-block:: console

    $ curl -F 'file=@hoge.jpg' localhost:8080/queue
    OK

サーバーログは以下のようになります。

.. code-block:: console

    2015-03-01 00:19:22.366  INFO 14014 --- [enerContainer-1] kanjava.App                              : received! GenericMessage [payload=byte[52075], headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[faceConverter], jms_priority=4, id=ba27919f-8758-58fc-9976-99262605295c, jms_timestamp=1425136762365, jms_expiration=0, jms_messageId=ID:2c46ab2c-bf5d-11e4-850e-eff6d41dec3e, timestamp=1425136762366}]
    2015-03-01 00:19:22.512  INFO 14014 --- [enerContainer-1] kanjava.FaceDetector                     : 1 faces are detected!

この処理では結果がわかりませんね。

次に50リクエストを同時に送ってみましょう。

.. code-block:: concole

    $ for i in `seq 1 50`;do curl -F 'file=@hoge.jpg' localhost:8080/queue; done
    OKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOKOK

全てレスポンスは返ってきています。サーバーログはどうでしょうか。

.. code-block:: console

    #
    # A fatal error has been detected by the Java Runtime Environment:
    #
    #  [thread 23815 also had an error]
    #
    # JRE version: Java(TM) SE Runtime Environment (8.0_20-b26) (build 1.8.0_20-b26)
    # Java VM: Java HotSpot(TM) 64-Bit Server VM (25.20-b23 mixed mode bsd-amd64 compressed oops)
    # Problematic frame:
    # C  [libopencv_objdetect.2.4.dylib+0xe307]  cv::HaarEvaluator::operator()(int) const+0x23
    #
    # Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
    #
    # An error report file with more information is saved as:
    # /xxxx/hs_err_pid14014.log
    #
    # If you would like to submit a bug report, please visit:
    #   http://bugreport.sun.com/bugreport/crash.jsp
    # The crash happened outside the Java Virtual Machine in native code.
    # See problematic frame for where to report the bug.
    #

JVMがハングしています・・・。

実は、\ :doc:`03-FaceConverterService`\ の段階でバグがありました。複数リクエストを同時に捌く際に起きているバグなので、
スレッドアンセーフによるバグですね。どこでしょうか。

JVMが落ちていることと、\ ``cv::HaarEvaluator::operator()(int)``\ がヒントです。OpenCVの顔検出部分が怪しいです。

以下のハイライト部分がスレッドアンセーフです。

.. code-block:: java
    :emphasize-lines: 6

    @JmsListener(destination = "faceConverter", concurrency = "1-5")
    void convertFace(Message<byte[]> message) throws IOException {
        log.info("received! {}", message);
        try (InputStream stream = new ByteArrayInputStream(message.getPayload())) {
            Mat source = Mat.createFrom(ImageIO.read(stream));
            faceDetector.detectFaces(source, FaceTranslator::duker); // この中の処理がスレッドアンセーフ!
            BufferedImage image = source.getBufferedImage();
            // do nothing...
        }
    }


正確には\ ``classifier.detectMultiScale(source, faceDetections);``\ の部分です。

\ ``classifier``\ がステートフルなため、\ ``FaceDetector``\ をデフォルトの\ ``singleton``\ スコープで登録しているのが問題なようです。

都度インスタンスを作り直す、\ ``prototype``\ スコープに変更しましょう。

以下のように、コンポーネントスキャン対象のクラスに\ ``@Scope``\ アノテーションをつけてスコープを明示します。

.. code-block:: java
    :emphasize-lines: 2

    @Component
    @Scope(value = "prototype")
    class FaceDetector {
        // ...
    }

実はこれだけでは、期待通りには動きません。Springではインスタンスのライフサイクルは寿命の長い方に合わせられます。

すなわち\ ``singleton``\ スコープの\ ``App``\ コントローラーに対して、\ ``prototype``\ スコープの\ ``FaceDetector``\ をインジェクションしても、
\ ``faceDetector``\ フィールドは寿命の長い\ ``singleton``\ スコープとして振る舞います。

この関係を変える(\ ``faceDetector``\ フィールドを\ ``prototype``\ スコープとして振る舞わせる)ために、scoped-proxyという仕組みを導入します。

\ ``@Scope``\ の\ ``proxyMode``\ 属性に以下のような設定を行います。

.. code-block:: java
    :emphasize-lines: 2

    @Component
    @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
    class FaceDetector {
        // ...
    }

これで\ ``FaceDetector``\ がProxyでラップされた状態で\ ``App``\ にインジェクションされるため、\ ``App``\ のスコープによらず、
\ ``faceDetector``\ フィールドは\ ``prototype``\ スコープでいられます。

この状態で\ ``App``\ クラスを再起動し、再度50リクエストを送ってみてください。\ ``FaceDetector``\ が毎回初期化され、無事全てのリクエストが捌かれているのがわかると思います。


.. note::

    \ ``FaceDetector``\ の初期化コストも大きいので、\ ``singleton``\ スコープのまま\ ``synchronized``\ による同期化を行っても良いです。
    どちらの性能が良いかは、サーバースペックと同時リクエスト数次第です。


本章では画像処理を非同期に実行しました。またインスタンスのスコープについて学びました。以上で本章は終了です。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc05」をつけてツイートしてください。

次は非同期に実行した処理結果を通知するために、STOMPという別のメッセージングプロトコルを使用します。
次章ではまずはSTOMPをつかってみましょう。

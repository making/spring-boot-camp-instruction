JMSを使ってみる
********************************************************************************

前章では画像処理サーバーをSpring MVCで作成しました。一般的に、画像処理のような重い処理を同期実行していくと、
同時リクエストによってリクエスト処理スレッドが枯渇しやすくなってしまいます。

そこで、画像処理リクエストを受けてもすぐに処理は行わずレスポンスだけ返し、実際の処理は非同期で行うことを考えましょう。

本章ではJMS(Java Message Service)を使用して、非同期プログラミングを試します。次章で画像処理サーバーのJMS対応を行います。

HTTPリクエストを受けたControllerは、すぐに本処理を行うのではなく、本処理に必要なデータを詰めたメッセージをJMSに対応したメッセージキュー製品に送信します。
メッセージ送信が完了すればHTTPレスポンスを返却します。

.. figure:: ./images/jms-send.png
    :width: 40%

送信されたメッセージは受信側によって取り出され、本処理がおこなれます。JMSによるメッセージ受信方法は色々あるのですが、今回はMessage Listenerを使用します。

.. figure:: ./images/jms-receive.png
    :width: 40%

本ハンズオンでは、メッセージキューとしてHornetQを使います。通常メッセージキューは別プロセスとして起動させますが、今回はセットアップの手間を省くため、
組み込みインメモリHornetQを使い、アプリと同じプロセス内で起動させます。

また、JMS APIを直接使わず、SpringのJMSサポート機能を使用します。

まずは、これまで作ったプロジェクトのpom.xmlに以下の依存関係を追加してください。

.. code-block:: xml

    <!-- SpringのJMSサポートとHornetQのクライアントライブラリを追加 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-hornetq</artifactId>
    </dependency>
    <!-- 組み込みインメモリHornetQ -->
    <dependency>
        <groupId>org.hornetq</groupId>
        <artifactId>hornetq-jms-server</artifactId>
    </dependency>

次に\ :file:`application.yml`\ に組み込みHornetQの設定を行います。


.. code-block:: yaml
    :emphasize-lines: 4-8

    spring:
      thymeleaf.cache: false
      main.show-banner: false
      hornetq:
        mode: embedded
        embedded:
          enabled: true
          queues: hello # 宛先名

この段階で\ ``App``\ クラスを実行してみてください。以下のようなログが出て、組み込みHornetQが起動しているのがわかります。

.. code-block:: console

    2015-02-28 20:33:14.431  INFO 13107 --- [           main] org.hornetq.core.server                  : HQ221000: live server is starting with configuration HornetQ Configuration (clustered=false,backup=false,sharedStore=true,journalDirectory=/var/folders/9p/hr0h11p124l0z7d3sqpvf5lw0000gn/T/hornetq-data/journal,bindingsDirectory=data/bindings,largeMessagesDirectory=data/largemessages,pagingDirectory=data/paging)
    2015-02-28 20:33:14.443  INFO 13107 --- [           main] org.hornetq.core.server                  : HQ221045: libaio is not available, switching the configuration into NIO
    2015-02-28 20:33:14.488  INFO 13107 --- [           main] org.hornetq.core.server                  : HQ221043: Adding protocol support CORE
    2015-02-28 20:33:14.558  INFO 13107 --- [           main] org.hornetq.core.server                  : HQ221003: trying to deploy queue jms.queue.hello
    2015-02-28 20:33:14.642  INFO 13107 --- [           main] org.hornetq.core.server                  : HQ221007: Server is now live
    2015-02-28 20:33:14.642  INFO 13107 --- [           main] org.hornetq.core.server                  : HQ221001: HornetQ Server version 2.4.5.FINAL (Wild Hornet, 124) [95281656-bf3d-11e4-aec8-496255b6cee1]

簡単なJMSプログラミングを行いましょう。まずは送信部分を作ります。

.. code-block:: java
    :emphasize-lines: 4-6,17,21-22,26-33

    package kanjava;

    // ...
    import org.springframework.jms.core.JmsMessagingTemplate;
    import org.springframework.messaging.Message;
    import org.springframework.messaging.support.MessageBuilder;

    // ...

    @SpringBootApplication
    @RestController
    public class App {
        public static void main(String[] args) {
            SpringApplication.run(App.class, args);
        }

        private static final Logger log = LoggerFactory.getLogger(App.class); // 後で使う

        @Autowired
        FaceDetector faceDetector;
        @Autowired
        JmsMessagingTemplate jmsMessagingTemplate; // メッセージ操作用APIのJMSラッパー

        // ...

        @RequestMapping(value = "/send")
        String send(@RequestParam String msg /* リクエストパラメータmsgでメッセージ本文を受け取る */) {
            Message<String> message = MessageBuilder
                    .withPayload(msg)
                    .build(); // メッセージを作成
            jmsMessagingTemplate.send("hello", message); // 宛先helloにメッセージを送信
            return "OK"; // とりあえずOKと即時応答しておく
        }
    }

これだけだと送りっぱなしで、メッセージを受け取り側がいません。次に受信部分(MessageListener)を書きましょう。

Spring 4.1からJMSのMessageListenerはとても書きやすくなり、Listenerとなるメソッドに\ ``@JmsListener``\ をつけるだけでよくなりました。
専用のクラスを作っても良いですし、Controllerの中に書いても有効です。

今回はシンプルにするため、\ ``App``\ クラス内にListenerメソッドを作成します。

.. code-block:: java
    :emphasize-lines: 4,21-25

    package kanjava;

    // ...
    import org.springframework.jms.annotation.JmsListener;
    // ...

    @SpringBootApplication
    @RestController
    public class App {
        // ...

        @RequestMapping(value = "/send")
        String send(@RequestParam String msg) {
            Message<String> message = MessageBuilder
                    .withPayload(msg)
                    .build();
            jmsMessagingTemplate.send("hello", message);
            return "OK";
        }

        @JmsListener(destination = "hello" /* 処理するメッセージの宛先を指定 */)
        void handleHelloMessage(Message<String> message /* 送信されたメッセージを受け取る */) {
            log.info("received! {}", message);
            log.info("msg={}", message.getPayload());
        }
    }

\ ``App``\ クラスを実行して、次のリクエストを送りましょう。

.. code-block:: console

    $ curl localhost:8080/send?msg=test
    OK

レスポンスが即返ってきました。サーバーログを確認しましょう。

.. code-block:: console

    2015-02-28 21:09:12.616  INFO 13367 --- [enerContainer-1] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=2eb99131-6f39-f6ca-9214-30221896c891, jms_timestamp=1425125352607, jms_expiration=0, jms_messageId=ID:9b87c38b-bf42-11e4-add4-7d91fe97ae4c, timestamp=1425125352615}]
    2015-02-28 21:09:12.616  INFO 13367 --- [enerContainer-1] kanjava.App                              : msg=test

メッセージキュー側のスレッド(スレッド名: DefaultMessageListenerContainer-スレッド数)で処理されているのがわかります。

デフォルトでは処理スレッド数は1です。スレッド数を変更する場合は\ ``@JmsListener``\ の\ ``concurrency``\ 属性を設定します。

.. code-block:: java

    @JmsListener(destination = "hello", concurrency = "1-5" /* 最小1スレッド、最大5スレッドに設定 */)

10リクエスト送ってみましょう。

.. code-block:: console

    $ for i in `seq 1 10`;do curl localhost:8080/send?msg=test;done
    OKOKOKOKOKOKOKOKOKOK

サーバーログは以下のようになります。

.. code-block:: console

    2015-02-28 21:19:47.655  INFO 13487 --- [enerContainer-1] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=76de8a0c-fd68-b059-8c91-5165e9845668, jms_timestamp=1425125987654, jms_expiration=0, jms_messageId=ID:160c3ea7-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987655}]
    2015-02-28 21:19:47.655  INFO 13487 --- [enerContainer-1] kanjava.App                              : msg=test
    2015-02-28 21:19:47.681  INFO 13487 --- [enerContainer-2] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=39870da1-e482-d98d-dba0-24702fa9cac9, jms_timestamp=1425125987680, jms_expiration=0, jms_messageId=ID:16105d5d-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987681}]
    2015-02-28 21:19:47.682  INFO 13487 --- [enerContainer-2] kanjava.App                              : msg=test
    2015-02-28 21:19:47.705  INFO 13487 --- [enerContainer-3] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=1f30f1b8-9c52-936a-5e8f-935f4f9db38b, jms_timestamp=1425125987703, jms_expiration=0, jms_messageId=ID:1613b8c3-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987705}]
    2015-02-28 21:19:47.705  INFO 13487 --- [enerContainer-3] kanjava.App                              : msg=test
    2015-02-28 21:19:47.729  INFO 13487 --- [enerContainer-4] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=4694e874-883b-8cb6-0dcf-09cf848bf9f1, jms_timestamp=1425125987727, jms_expiration=0, jms_messageId=ID:16176249-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987729}]
    2015-02-28 21:19:47.729  INFO 13487 --- [enerContainer-4] kanjava.App                              : msg=test
    2015-02-28 21:19:47.751  INFO 13487 --- [enerContainer-5] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=93c2b05a-6350-4cca-1bf7-2cb6e8e6fef8, jms_timestamp=1425125987749, jms_expiration=0, jms_messageId=ID:161abdaf-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987751}]
    2015-02-28 21:19:47.751  INFO 13487 --- [enerContainer-5] kanjava.App                              : msg=test
    2015-02-28 21:19:47.775  INFO 13487 --- [enerContainer-1] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=398007e6-70dd-4fef-1954-443fc82269c4, jms_timestamp=1425125987773, jms_expiration=0, jms_messageId=ID:161e6735-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987774}]
    2015-02-28 21:19:47.775  INFO 13487 --- [enerContainer-1] kanjava.App                              : msg=test
    2015-02-28 21:19:47.803  INFO 13487 --- [enerContainer-2] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=78080611-f326-a5b7-5737-6ff93c58c4b3, jms_timestamp=1425125987800, jms_expiration=0, jms_messageId=ID:162285eb-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987803}]
    2015-02-28 21:19:47.803  INFO 13487 --- [enerContainer-2] kanjava.App                              : msg=test
    2015-02-28 21:19:47.835  INFO 13487 --- [enerContainer-3] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=a28ddffb-e0ef-a2f9-8b09-5248f78d23d5, jms_timestamp=1425125987834, jms_expiration=0, jms_messageId=ID:1627b611-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987835}]
    2015-02-28 21:19:47.836  INFO 13487 --- [enerContainer-3] kanjava.App                              : msg=test
    2015-02-28 21:19:47.858  INFO 13487 --- [enerContainer-4] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=641fc649-d29c-ef42-b564-77985b96eb05, jms_timestamp=1425125987856, jms_expiration=0, jms_messageId=ID:162b1177-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987858}]
    2015-02-28 21:19:47.858  INFO 13487 --- [enerContainer-4] kanjava.App                              : msg=test
    2015-02-28 21:19:47.886  INFO 13487 --- [enerContainer-5] kanjava.App                              : received! GenericMessage [payload=test, headers={jms_redelivered=false, jms_deliveryMode=2, JMSXDeliveryCount=1, jms_destination=HornetQQueue[hello], jms_priority=4, id=43dabfa3-3f07-a75b-41b9-1bfed1b716fb, jms_timestamp=1425125987884, jms_expiration=0, jms_messageId=ID:162f573d-bf44-11e4-a10f-3f034703c26f, timestamp=1425125987886}]
    2015-02-28 21:19:47.887  INFO 13487 --- [enerContainer-5] kanjava.App                              : msg=test

5スレッドで処理されていることがわかります。

.. note::

    本章で使用した\ ``JmsMessagingTemplate``\ は、昔からあるSpringのJMS APIラッパーである\ ``JmsTemplate``\ をメッセージング抽象化機構でさらにラップしたものです。Spring 4.1から追加されました。
    メッセージ操作の用のシグニチャ(\ ``MessageSendingOperations``\ など)や送信する\ ``Message``\ クラスはJMSに限らず、次に説明するSTOMPなどSpringのメッセージング関連のプログラミングで使用できます。

    Spring 4.1で追加されたJMS連携機能やspring-messagingについてはこちらの\ `資料 <http://www.slideshare.net/makingx/springone-2gx-2014-spring-41-jsug/44>`_\ を参照してください。


JMSの簡単な使い方を学びました。以上で本章は終了です。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc04」をつけてツイートしてください。

次は顔変換処理をMessageListenerで行うようにします。




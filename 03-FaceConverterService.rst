顔変換サービスの作成
********************************************************************************
本章では「\ :doc:`01-HelloWorld`\ 」と「\ :doc:`02-OpenCV`\ 」で作成した内容を統合して、顔変換Webサービスを作成します。

まずは「\ :doc:`01-HelloWorld`\ 」で作成したpom.xmlに、「\ :doc:`02-OpenCV`\ 」の内容を追記します。

以下のハイライトされた文を追加してください。

.. code-block:: xml
    :emphasize-lines: 23-24,36-51,136-185

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>

        <groupId>kanjava</groupId>
        <artifactId>kusokora</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <packaging>jar</packaging>

        <name>Spring Boot Docker Blank Project (from https://github.com/making/spring-boot-docker-blank)</name>

        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>1.2.1.RELEASE</version>
        </parent>

        <properties>
            <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
            <start-class>kanjava.App</start-class>
            <java.version>1.8</java.version>
            <javacv.version>0.10</javacv.version>
            <opencv.version>2.4.10-${javacv.version}</opencv.version>
        </properties>

        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
            </dependency>
            <dependency>
                <groupId>org.bytedeco</groupId>
                <artifactId>javacv</artifactId>
                <version>${javacv.version}</version>
            </dependency>
            <dependency>
                <groupId>org.bytedeco.javacpp-presets</groupId>
                <artifactId>opencv</artifactId>
                <version>${opencv.version}</version>
            </dependency>
            <dependency>
                <groupId>org.bytedeco.javacpp-presets</groupId>
                <artifactId>opencv</artifactId>
                <version>${opencv.version}</version>
                <classifier>${classifier}</classifier>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
        </dependencies>
        <build>
            <finalName>${project.artifactId}</finalName>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <dependencies>
                        <dependency>
                            <groupId>org.springframework</groupId>
                            <artifactId>springloaded</artifactId>
                            <version>${spring-loaded.version}</version>
                        </dependency>
                    </dependencies>
                </plugin>

                <!-- Copy Dockerfile -->
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>copy-resources</id>
                            <phase>validate</phase>
                            <goals>
                                <goal>copy-resources</goal>
                            </goals>
                            <configuration>
                                <outputDirectory>${basedir}/target/</outputDirectory>
                                <resources>
                                    <resource>
                                        <directory>src/main/docker</directory>
                                        <filtering>true</filtering>
                                    </resource>
                                </resources>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>com.coderplus.maven.plugins</groupId>
                    <artifactId>copy-rename-maven-plugin</artifactId>
                    <version>1.0</version>
                    <executions>
                        <execution>
                            <id>rename-file</id>
                            <phase>validate</phase>
                            <goals>
                                <goal>rename</goal>
                            </goals>
                            <configuration>
                                <sourceFile>${basedir}/target/Dockerfile.txt</sourceFile>
                                <destinationFile>${basedir}/target/Dockerfile</destinationFile>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-antrun-plugin</artifactId>
                    <version>1.7</version>
                    <executions>
                        <execution>
                            <id>zip-files</id>
                            <phase>package</phase>
                            <goals>
                                <goal>run</goal>
                            </goals>
                            <configuration>
                                <target>
                                    <zip destfile="${basedir}/target/app.zip" basedir="${basedir}/target"
                                         includes="Dockerfile, Dockerrun.aws.json, ${project.artifactId}.jar"/>
                                </target>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>

        <profiles>
            <profile>
                <id>macosx-x86_64</id>
                <activation>
                    <os>
                        <family>mac</family>
                        <arch>x86_64</arch>
                    </os>
                </activation>
                <properties>
                    <classifier>macosx-x86_64</classifier>
                </properties>
            </profile>
            <profile>
                <id>linux-x86_64</id>
                <activation>
                    <os>
                        <family>unix</family>
                        <arch>amd64</arch>
                    </os>
                </activation>
                <properties>
                    <classifier>linux-x86_64</classifier>
                </properties>
            </profile>
            <profile>
                <id>windows-x86_64</id>
                <activation>
                    <os>
                        <family>windows</family>
                        <arch>amd64</arch>
                    </os>
                </activation>
                <properties>
                    <classifier>windows-x86_64</classifier>
                </properties>
            </profile>
            <profile>
                <id>windows-x86</id>
                <activation>
                    <os>
                        <family>windows</family>
                        <arch>x86</arch>
                    </os>
                </activation>
                <properties>
                    <classifier>windows-x86</classifier>
                </properties>
            </profile>
        </profiles>
    </project>


次に「\ :doc:`01-HelloWorld`\ 」で作成した\ ``App``\ クラスに、「\ :doc:`02-OpenCV`\ 」で作成した顔変換処理を移植します。

「\ :doc:`02-OpenCV`\ 」では1メソッドにベタ書きしたので、今回は以下のように顔検出処理と顔変換処理を分けて、それぞれ別クラスに定義します。

.. code-block:: java
    :emphasize-lines: 3-4,7,11,26-37

    package kanjava;

    import static org.bytedeco.javacpp.opencv_core.*;
    import static org.bytedeco.javacpp.opencv_objdetect.*;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.stereotype.Component;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

    import java.util.function.BiConsumer;

    @SpringBootApplication
    @RestController
    public class App {
        public static void main(String[] args) {
            SpringApplication.run(App.class, args);
        }

        @RequestMapping(value = "/")
        String hello() {
            return "Hello World!";
        }
    }

    @Component // コンポーネントスキャン対象にする。@Serviceでも@NamedでもOK
    class FaceDetector {
        public void detectFaces(Mat source /* 入力画像 */, BiConsumer<Mat, Rect> detectAction /* 顔領域に対応する処理 */) {
            // ここに顔検出処理を実装する
        }
    }

    class FaceTranslator {
        public static void duker(Mat source, Rect r) { // Duke化するメソッド
            // ここに顔変換処理を実装する
        }
    }


実際の処理を埋めましょう。

.. code-block:: java
    :emphasize-lines: 3-5,12-14,35-67,72-82

    package kanjava;

    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.stereotype.Component;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

    import javax.annotation.PostConstruct;
    import java.io.File;
    import java.io.IOException;
    import java.util.function.BiConsumer;

    import static org.bytedeco.javacpp.opencv_core.*;
    import static org.bytedeco.javacpp.opencv_objdetect.*;

    @SpringBootApplication
    @RestController
    public class App {
        public static void main(String[] args) {
            SpringApplication.run(App.class, args);
        }

        @RequestMapping(value = "/")
        String hello() {
            return "Hello World!";
        }
    }

    @Component
    class FaceDetector {
        // 分類器のパスをプロパティから取得できるようにする
        @Value("${classifierFile:classpath:/haarcascade_frontalface_default.xml}")
        File classifierFile;

        CascadeClassifier classifier;

        static final Logger log = LoggerFactory.getLogger(FaceDetector.class);

        public void detectFaces(Mat source, BiConsumer<Mat, Rect> detectAction) {
            // 顔認識結果
            Rect faceDetections = new Rect();
            // 顔認識実行
            classifier.detectMultiScale(source, faceDetections);
            // 認識した顔の数
            int numOfFaces = faceDetections.limit();
            log.info("{} faces are detected!", numOfFaces);
            for (int i = 0; i < numOfFaces; i++) {
                // i番目の認識結果
                Rect r = faceDetections.position(i);
                // 1件ごとの認識結果を変換処理(関数)にかける
                detectAction.accept(source, r);
            }
        }

        @PostConstruct // 初期化処理。DIでプロパティがセットされたあとにclassifierインスタンスを生成したいのでここで書く。
        void init() throws IOException {
            if (log.isInfoEnabled()) {
                log.info("load {}", classifierFile.toPath());
            }
            // 分類器の読み込み
            this.classifier = new CascadeClassifier(classifierFile.toPath()
                    .toString());
        }
    }

    class FaceTranslator {
        public static void duker(Mat source, Rect r) { // BiConsumer<Mat, Rect>で渡せるようにする
            int x = r.x(), y = r.y(), h = r.height(), w = r.width();
            // Dukeのように描画する
            // 上半分の黒四角
            rectangle(source, new Point(x, y), new Point(x + w, y + h / 2),
                    new Scalar(0, 0, 0, 0), -1, CV_AA, 0);
            // 下半分の白四角
            rectangle(source, new Point(x, y + h / 2), new Point(x + w, y + h),
                    new Scalar(255, 255, 255, 0), -1, CV_AA, 0);
            // 中央の赤丸
            circle(source, new Point(x + h / 2, y + h / 2), (w + h) / 12,
                    new Scalar(0, 0, 255, 0), -1, CV_AA, 0);
        }
    }

次に、この画像処理ロジックをControllerから叩きます。処理結果の画像をレスポンスとして返すのにJavaCVから扱いやすい\ ``BufferedImage``\ をそのままシリアライズさせましょう。
\ ``BufferedImage``\ のシリアライズはSpring Bootのデフォルトでは対応していないのですが、特定の型に対するリクエスト・レスポンスを扱うための\ ``HttpMessageConverter``\ の\ ``BufferedImage``\
は用意されています。\ ``org.springframework.http.converter.BufferedImageHttpMessageConverter``\ です。

Spring Bootで新しい\ ``HttpMessageConverter``\ を追加したい場合、対象の\ ``HttpMessageConverter``\ をBean定義するだけで良いです。

Spring BootでBean定義する場合は通常、\ ``@Bean``\ を使ってJavaで定義します。\ ``@Configuration``\  (またはそれを内包する\ ``@SpringBootApplication``\ ) がついたクラスの中で、
インスタンスを生成するメソッドを書き、そのメソッドに\ ``@Bean``\ をつければ良いです。

今回の場合、以下のようになります。

.. code-block:: java

    @Bean
    BufferedImageHttpMessageConverter bufferedImageHttpMessageConverter() {
        return new BufferedImageHttpMessageConverter();
    }

それでは画像変換を行うControllerの処理を追加しましょう。

.. code-block:: java
    :emphasize-lines: 5,9-10,13-14,18-20,35-41,48-55

    package kanjava;

    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.context.annotation.Bean;
    import org.springframework.http.converter.BufferedImageHttpMessageConverter;
    import org.springframework.stereotype.Component;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RequestMethod;
    import org.springframework.web.bind.annotation.RequestParam;
    import org.springframework.web.bind.annotation.RestController;

    import javax.annotation.PostConstruct;
    import javax.imageio.ImageIO;
    import javax.servlet.http.Part;
    import java.awt.image.BufferedImage;
    import java.io.File;
    import java.io.IOException;
    import java.util.function.BiConsumer;

    import static org.bytedeco.javacpp.opencv_core.*;
    import static org.bytedeco.javacpp.opencv_objdetect.*;

    @SpringBootApplication
    @RestController
    public class App {
        public static void main(String[] args) {
            SpringApplication.run(App.class, args);
        }

        @Autowired // FaceDetectorをインジェクション
        FaceDetector faceDetector;

        @Bean // HTTPのリクエスト・レスポンスボディにBufferedImageを使えるようにする
        BufferedImageHttpMessageConverter bufferedImageHttpMessageConverter() {
            return new BufferedImageHttpMessageConverter();
        }

        @RequestMapping(value = "/")
        String hello() {
            return "Hello World!";
        }

        // curl -v -F 'file=@hoge.jpg' http://localhost:8080/duker > after.jpg という風に使えるようにする
        @RequestMapping(value = "/duker", method = RequestMethod.POST) // POSTで/dukerへのリクエストに対する処理
        BufferedImage duker(@RequestParam Part file /* パラメータ名fileのマルチパートリクエストのパラメータを取得 */) throws IOException {
            Mat source = Mat.createFrom(ImageIO.read(file.getInputStream())); // Part -> BufferedImage -> Matと変換
            faceDetector.detectFaces(source, FaceTranslator::duker); // 対象のMatに対して顔認識。認識結果に対してduker関数を適用する。
            BufferedImage image = source.getBufferedImage(); // Mat -> BufferedImage
            return image;
        }
    }

    @Component
    class FaceDetector {
        @Value("${classifierFile:classpath:/haarcascade_frontalface_default.xml}")
        File classifierFile;

        CascadeClassifier classifier;

        static final Logger log = LoggerFactory.getLogger(FaceDetector.class);

        public void detectFaces(Mat source, BiConsumer<Mat, Rect> detectAction) {
            // 顔認識結果
            Rect faceDetections = new Rect();
            // 顔認識実行
            classifier.detectMultiScale(source, faceDetections);
            // 認識した顔の数
            int numOfFaces = faceDetections.limit();
            log.info("{} faces are detected!", numOfFaces);
            for (int i = 0; i < numOfFaces; i++) {
                // i番目の認識結果
                Rect r = faceDetections.position(i);
                // 認識結果を変換処理にかける
                detectAction.accept(source, r);
            }
        }

        @PostConstruct
        void init() throws IOException {
            if (log.isInfoEnabled()) {
                log.info("load {}", classifierFile.toPath());
            }
            // 分類器の読み込み
            this.classifier = new CascadeClassifier(classifierFile.toPath()
                    .toString());
        }
    }

    class FaceTranslator {
        public static void duker(Mat source, Rect r) {
            int x = r.x(), y = r.y(), h = r.height(), w = r.width();
            // Dukeのように描画する
            // 上半分の黒四角
            rectangle(source, new Point(x, y), new Point(x + w, y + h / 2),
                    new Scalar(0, 0, 0, 0), -1, CV_AA, 0);
            // 下半分の白四角
            rectangle(source, new Point(x, y + h / 2), new Point(x + w, y + h),
                    new Scalar(255, 255, 255, 0), -1, CV_AA, 0);
            // 中央の赤丸
            circle(source, new Point(x + h / 2, y + h / 2), (w + h) / 12,
                    new Scalar(0, 0, 255, 0), -1, CV_AA, 0);
        }
    }


実行する前に、「\ :doc:`02-OpenCV`\ 」で使用した\ :file:`haarcascade_frontalface_default.xml`\ を\ :file:`src/main/resources`\ にコピーしましょう。以下のように\ ``wget``\ しても構いません。

.. code-block:: console

    $ wget https://github.com/making/hello-cv/raw/duker/src/main/resources/haarcascade_frontalface_default.xml


ファイルをコピーしたら、早速起動しましょう。

.. code-block:: console

    $ mvn spring-boot:run

\ ``main``\ メソッド実行でも構いません。

顔画像を以下のように送ってください。

.. code-block:: console

    $ curl -v -F 'file=@hoge.jpg' http://localhost:8080/duker > after.jpg

画像のフォーマットが認識されない場合は、リクエストパスに拡張子をつけてメディアタイプを明示してください。

.. code-block:: console

    $ curl -v -F 'file=@hoge.jpg' http://localhost:8080/duker.jpg > after.jpg

変換後の\ :file:`after.jpg`\ を開いてください。顔がduke化されていますか？


余裕があれば、\ ``FaceTranslator``\ に独自の顔変換ロジックを書いてみましょう。

.. code-block:: java

    class FaceTranslator {
        // ...

        public static void kusokora(Mat source, Rect r) {
            // 変換処理
        }
    }

Controllerにも以下のメソッドを追加しましょう。

.. code-block:: java

    @RequestMapping(value = "/kusokora", method = RequestMethod.POST)
    BufferedImage kusokora(@RequestParam Part file) throws IOException {
        Mat source = Mat.createFrom(ImageIO.read(file.getInputStream()));
        faceDetector.detectFaces(source, FaceTranslator::kusokora);
        BufferedImage image = source.getBufferedImage();
        return image;
    }

以上で本章は終了です。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc03」をつけてツイートしてください。
是非変換後の画像もつけてツイートしてください。


.. warning::

    実はこのクラス(Controller)にはバグがあります。「\ :doc:`05-AsyncFaceConverter`\ 」で修正しますが、問題に気づきましたか？

次はこの顔変換処理を非同期で行うようにします。次章ではその前段として、Spring BootでJMSを使う方法を学びます。
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
    :emphasize-lines: 3-4,7,26-37

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



本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc03」をつけてツイートしてください。
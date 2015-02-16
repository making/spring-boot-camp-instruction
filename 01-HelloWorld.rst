[事前準備] Spring BootでHello World
********************************************************************************

Spring BootプロジェクトをMaven Archetypeから作ります。今回は拙作の\ `spring-boot-docker-blank <https://github.com/making/spring-boot-docker-blank>`_\ を使用します。
このMaven ArchetypeにはDockerデプロイするための設定が予め行われています。

以下のコマンドでプロジェクトを作成しましょう。ターミナルまたはコマンドプロンプトに貼付けてください。

* Bashを使っている場合

  .. code-block:: console

    $ mvn archetype:generate -B\
     -DarchetypeGroupId=am.ik.archetype\
     -DarchetypeArtifactId=spring-boot-docker-blank-archetype\
     -DarchetypeVersion=1.0.2\
     -DgroupId=kanjava\
     -DartifactId=kusokora\
     -Dversion=1.0.0-SNAPSHOT
    $ cd kusokora


* コマンドプロンプトを使っている場合

  .. code-block:: console

    $ mvn archetype:generate -B^
     -DarchetypeGroupId=am.ik.archetype^
     -DarchetypeArtifactId=spring-boot-docker-blank-archetype^
     -DarchetypeVersion=1.0.2^
     -DgroupId=kanjava^
     -DartifactId=kusokora^
     -Dversion=1.0.0-SNAPSHOT
    $ cd kusokora

生成されたプロジェクトは以下のような構造になっています。

.. code-block:: console

    kusokora/
    ├── pom.xml
    └── src
        ├── main
        │   ├── docker ... Docker用ファイル格納フォルダ。(Dockerデプロイするときのみ使う)
        │   │   ├── Dockerfile.txt
        │   │   └── Dockerrun.aws.json
        │   ├── java
        │   │   └── kanjava
        │   │       └── App.java ... アプリケーションコードを書くJavaファイル。今回のハンズオンでは基本的にこのファイルしか使わない。
        │   └── resources
        │       └── application.yml ... Spring Bootの設定ファイル。無くても良い。
        └── test ... テスト用フォルダ。今回は使わない。
            ├── java
            └── resources


以下のようなpom.xmlになっています。簡単に説明を加えたので、気になる人は見ておいてください。

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>

        <groupId>kanjava</groupId>
        <artifactId>kusokora</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <packaging>jar</packaging>

        <name>Spring Boot Docker Blank Project (from https://github.com/making/spring-boot-docker-blank)</name>

        <!-- 最重要。Spring Bootの諸々設定を引き継ぐための親情報。 -->
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>1.2.1.RELEASE</version>
        </parent>

        <properties>
            <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
            <start-class>kanjava.App</start-class><!-- mainメソッドのあるクラスを明示的に指定 -->
            <java.version>1.8</java.version>
        </properties>

        <dependencies>
            <!-- 重要。Webアプリをつくるための設定。必要な依存関係は実はこれだけ。 -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <!-- メトリクスや環境変数を返すエンドポイントの設定。ここはおまけ。 -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
            </dependency>
            <!-- テストの設定。今回はテストしないので、ここはおまけ。 -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
        </dependencies>
        <build>
            <finalName>${project.artifactId}</finalName>
            <plugins>
                <!-- Spring Bootプラグインの設定(必須)。Spring Loadedも設定している。 -->
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

                <!-- ここから下はDocker用のちょっとした設定で本質的でない。無視しても良い。 -->
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
                <!-- ほんとどうでもいい設定。 -->
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
                <!-- AWS Elastic BeanStalk用のzipを作成。ここも本質的でない。 -->
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
                                    <zip destfile="${basedir}/target/app.zip" basedir="${basedir}/target" includes="Dockerfile, Dockerrun.aws.json, ${project.artifactId}.jar" />
                                </target>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </project>

src/main/java/kanjava/App.javaを見てください。

.. code-block:: java

    package kanjava;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

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

\ ``@SpringBootApplication``\ が魔法のアノテーションです。このアノテーションは以下の3アノテーションを1つにまとめたものです。

.. tabularcolumns:: |p{0.10\linewidth}|p{0.90\linewidth}|
.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - アノテーション
     - 説明
   * - | \ ``@EnableAutoConfiguration``\
     - | Spring Bootの自動設定群を有効にします。
   * - | \ ``@ComponentScan``\
     - | コンポーネントスキャンを行う。このクラスのパッケージ配下で\ ``@Component``\ , \ ``@Service``\ , \ ``@Repository``\ , \ ``@Controller``\ , \ ``@RestController``\ , \ ``@Configuration``\ ,\ ``@Named``\ つきのクラスをDIコンテナに登録します。
   * - | \ ``@Configuration``\
     - | このクラス自体をBean定義可能にします。\ ``@Bean``\ をつけたメソッドをこのクラス内に定義することで、DIコンテナにBeanを登録できます。


\ ``@RestController``\ をつけることで、このクラス自体がSpring MVCのコントローラーになります。
このアノテーションをつけたクラスのメソッドに\ ``@RequestMapping``\ をつけるとリクエストを受けるメソッドになり、そのメソッドの返り値がレスポンスボディに書き込まれます。

この例だと、"/"にアクセスすると\ ``hello()``\ メソッドが呼ばれ、"Hello World!"がレスポンスボディに書き込まれます。Content-Typeは"text/plain"になります。

\ ``main``\ メソッドを見てください。\ ``SpringApplication.run(App.class, args)``\ がSpring Bootアプリケーションを起動するメソッドです。

このmainメソッドをIDEから実行してみてください。Tomcatが立ち上がり、8080番ポートがlistenされます。すでに8080番ポートが使用されている場合は、起動に失敗するので使用しているプロセスを終了させてください。

http://localhost:8080\ にアクセスしてください。「Hello World!」が表示されましたか？

次にMavenプラグインから実行してみましょう。

.. code-block:: console

    $ mvn spring-boot:run

同様に起動しますね。


.. note::

    この雛形プロジェクトには"Spring Boot Actuator"が設定されており、環境変数やメトリクス、ヘルスチェックなど非機能面のサポートが初めからされています。
    次のURLにアクセスして、色々な情報を取得してみてください。

    * http://localhost:8080/env
    * http://localhost:8080/health
    * http://localhost:8080/configprops
    * http://localhost:8080/mappings
    * http://localhost:8080/metrics
    * http://localhost:8080/beans
    * http://localhost:8080/trace
    * http://localhost:8080/dump
    * http://localhost:8080/info

    Chromeを利用している場合は、\ `JSONView <https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc>`_\ をインストールしておくと便利です。


今度は実行可能jarを作ります。

.. code-block:: console

    $ mvn clean package

targetの下にkusokora.jarが出来ています。これを実行してください。

.. code-block:: console

    $ mvn -jar target/kusokora.jar

これも同様に起動します。


ちなみに、ポート番号を変えるときは

.. code-block:: console

    $ mvn spring-boot:run -Drun.arguments="--server.port=9999"

や

.. code-block:: console

    $ mvn -jar target/kusokora.jar --server.port=9999

で指定できます。今度は\ http://localhost:9999\ にアクセスできます。

最後にsrc/main/resources/application.ymlを見てください。以下の設定がされています。
よく使うものが予め設定されていますが、今回は特に必要ではありません。気になるようであれば削除してください。ファイルごと消しても構いません。

.. code-block:: yaml

    # See http://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html
    spring:
      thymeleaf.cache: false # Thymeleafを使ったときにテンプレートをキャッシュさせない(開発用)
      main.show-banner: false # 起動時にバナー表示をOFFにする


.. note::

    Dockerデプロイも試したい場合は、「\ :doc:`10-Docker`\ 」を先にみてください。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc01」をつけてツイートしてください。
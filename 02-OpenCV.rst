[事前準備] JavaでOpenCVを使う
********************************************************************************

画像処理を行うために、超有名なライブラリである\ `OpenCV <http://opencv.org/>`_\ を使用します。JavaからOpenCVを扱うために、今回は\ `JavaCV <https://github.com/bytedeco/javacv>`_\ というライブラリを使います。

JavaCVは\ `JavaCPP <https://github.com/bytedeco/javacpp>`_\ というC++のソースから自動生成してできるブリッジのようなもので作られています。MavenやGradleなどの依存性解決の仕組みで簡単に利用できて、手軽にセットアップできるので今回採用しました。

本章では、手元の環境でJavaCVを利用できるかどうかを確認します。

まずは先ほどのkusokoraディレクトリの1つ上の階層に移動してください。

.. code-block:: console

    $ cd ..

そして、サンプルプロジェクトをcloneします。

.. code-block:: console

    $ git clone https://github.com/making/hello-cv.git
    $ cd hello-cv

チェックアウトしたプロジェクトのpom.xmlの後半の部分を見てください。

.. code-block:: xml

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

実行環境により、どのプロファイル(アーキテクチャ)を使用するかを判断しています。このプロファイルで定義されている\ ``<classifier>``\ プロパティが、

.. code-block:: xml

    <dependency>
        <groupId>org.bytedeco.javacpp-presets</groupId>
        <artifactId>opencv</artifactId>
        <version>${opencv.version}</version>
        <classifier>${classifier}</classifier>
    </dependency>

に使われ、環境にあったネイティブライブラリをダウンロードします。

では早速サンプルアプリを実行してみましょう。



.. note::

    このサンプルアプリには\ `このブログ記事 <http://blog.ik.am/#/entries/318>`_\ に対応し、master以外に2つのブランチが含まれています。興味があればブランチを切り替えてサンプルを実行してみてください。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc02」をつけてツイートしてください。
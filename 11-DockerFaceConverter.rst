顔変換サービスをDocker化
********************************************************************************

\ :file:`src/main/docker/Dockerfile.txt`\ をみてください。

.. code-block:: bash

    FROM dockerfile/java:oracle-java8

    ADD kusokora.jar /opt/kusokora/
    EXPOSE 8080
    WORKDIR /opt/kusokora/
    CMD ["java", "-Xms512m", "-Xmx1g", "-jar", "kusokora.jar"]

実行可能jarをコピーして実行しているだけです。通常はこれでOKなのですが、今回はOpenCVを使うため、OpenCV用の準備が必要です。


\ :file:`src/main/docker/Dockerfile.txt`\ 以下のように書き換えてください。

.. code-block:: bash

    FROM centos:centos7
    RUN yum -y update && yum clean all
    RUN yum -y install wget glibc gtk2 gstreamer gstreamer-plugins-base libv4l
    RUN wget -c -O /tmp/jdk-8u31-linux-x64.rpm --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u31-b13/jdk-8u31-linux-x64.rpm
    RUN yum -y localinstall /tmp/jdk-8u31-linux-x64.rpm
    RUN rm -f /tmp/jdk-8u31-linux-x64.rpm

    ADD kusokora.jar /opt/kusokora/
    ADD classes/haarcascade_frontalface_default.xml /opt/kusokora/
    EXPOSE 8080
    WORKDIR /opt/kusokora/
    CMD ["java", "-Xms512m", "-Xmx1g", "-jar", "kusokora.jar", "--classifierFile=haarcascade_frontalface_default.xml"]

また、JavaCVの制限でhaarcascade_frontalface_default.xmlをjarの中から読み込めないため、外出しして渡す必要があります。
haarcascade_frontalface_default.xmlをAWSデプロイ用のzipに含めるためにpom.xmlの以下の部分を修正します。


.. code-block:: xml
    :emphasize-lines: 9-10

    <execution>
        <id>zip-files</id>
        <phase>package</phase>
        <goals>
            <goal>run</goal>
        </goals>
        <configuration>
            <target>
                <!-- classes/haarcascade_frontalface_default.xmlを追加 -->
                <zip destfile="${basedir}/target/app.zip" basedir="${basedir}/target" includes="Dockerfile, Dockerrun.aws.json, ${project.artifactId}.jar, classes/haarcascade_frontalface_default.xml" />
            </target>
        </configuration>
    </execution>

Docker上はLinux(CentOS 7)を使用するので、Linux用にアプリケーションをビルドします。

.. code-block:: console

    $ mvn clean package -Plinux-x86_64

\ :file:`target`\ 以下に\ :file:`app.zip`\ ができるので、これを「\ :doc:`10-Docker`\ 」のように、AWSにデプロイしたり、
ローカルの\ ``boot2docker``\ で試すなりしてみてください。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc11」をつけてツイートしてください。

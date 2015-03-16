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

Docker上はLinux(CentOS 7)を使用するので、Linux用にアプリケーションをビルドします。

.. code-block:: console

    $ mvn clean package -Plinux-x86_64

\ :file:`target`\ 以下に\ :file:`app.zip`\ ができるので、これを「\ :doc:`10-Docker`\ 」のように、AWSにデプロイしたり、
ローカルの\ ``boot2docker``\ で試すなりしてみてください。

本章の内容を修了したらハッシュタグ「#kanjava_sbc #sbc11」をつけてツイートしてください。

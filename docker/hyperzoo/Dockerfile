ARG SPARK_VERSION=2.4.3
ARG SPARK_HOME=/opt/spark
ARG JDK_VERSION=8u192
ARG JDK_URL=your_jdk_url
ARG BIGDL_VERSION=0.12.2
ARG ANALYTICS_ZOO_VERSION=0.11.0-SNAPSHOT
ARG TINI_VERSION=v0.18.0

# stage.1 jdk & spark
FROM ubuntu:18.04 as spark
ARG SPARK_VERSION
ARG JDK_VERSION
ARG JDK_URL
ARG SPARK_HOME
ENV TINI_VERSION                        v0.18.0
ENV SPARK_VERSION                       ${SPARK_VERSION}
ENV SPARK_HOME                          ${SPARK_HOME}
RUN apt-get update --fix-missing && \
    apt-get install -y apt-utils vim curl nano wget unzip maven git && \
# java
    wget $JDK_URL && \
    gunzip jdk-$JDK_VERSION-linux-x64.tar.gz && \
    tar -xf jdk-$JDK_VERSION-linux-x64.tar -C /opt && \
    rm jdk-$JDK_VERSION-linux-x64.tar && \
    mv /opt/jdk* /opt/jdk$JDK_VERSION && \
    ln -s /opt/jdk$JDK_VERSION /opt/jdk && \
# spark
    wget https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop2.7.tgz && \
    tar -zxvf spark-${SPARK_VERSION}-bin-hadoop2.7.tgz && \
    mv spark-${SPARK_VERSION}-bin-hadoop2.7 /opt/spark && \
    rm spark-${SPARK_VERSION}-bin-hadoop2.7.tgz && \
    cp /opt/spark/kubernetes/dockerfiles/spark/entrypoint.sh /opt && \
    rm $SPARK_HOME/jars/kubernetes-client-*.jar

ADD https://repo1.maven.org/maven2/io/fabric8/kubernetes-client/4.4.2/kubernetes-client-4.4.2.jar $SPARK_HOME/jars
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /sbin/tini

# stage.2 analytics-zoo
FROM ubuntu:18.04 as analytics-zoo
ARG SPARK_VERSION
ARG BIGDL_VERSION
ARG ANALYTICS_ZOO_VERSION

ENV SPARK_VERSION		${SPARK_VERSION}
ENV BIGDL_VERSION		${BIGDL_VERSION}
ENV ANALYTICS_ZOO_VERSION               ${ANALYTICS_ZOO_VERSION}
ENV ANALYTICS_ZOO_HOME                  /opt/analytics-zoo-${ANALYTICS_ZOO_VERSION}

RUN apt-get update --fix-missing && \
    apt-get install -y apt-utils vim curl nano wget unzip maven git
ADD ./download-analytics-zoo.sh /opt

RUN chmod a+x /opt/download-analytics-zoo.sh && \
    mkdir -p /opt/analytics-zoo-examples/python
RUN /opt/download-analytics-zoo.sh && \
    rm analytics-zoo-bigdl*.zip && \
    unzip $ANALYTICS_ZOO_HOME/lib/*.zip 'zoo/examples/*' -d /opt/analytics-zoo-examples/python && \
    mv /opt/analytics-zoo-examples/python/zoo/examples/* /opt/analytics-zoo-examples/python && \
    rm -rf /opt/analytics-zoo-examples/python/zoo/examples

# stage.3 redis
FROM ubuntu:18.04 as redisenv
#redis
RUN apt-get update --fix-missing && \
    apt-get install -y apt-utils wget && \
    apt-get install -y gcc g++ make
RUN wget http://download.redis.io/releases/redis-5.0.5.tar.gz && \
    tar xzf redis-5.0.5.tar.gz -C /opt && \
    rm redis-5.0.5.tar.gz && \
    cd /opt/redis-5.0.5 && \
    make && \
    echo "bind 0.0.0.0" >> /opt/redis-5.0.5/redis.conf && \
    cd ..

# stage.4 
FROM ubuntu:18.04
MAINTAINER The Analytics-Zoo Authors https://github.com/intel-analytics/analytics-zoo
ARG ANALYTICS_ZOO_VERSION
ARG BIGDL_VERSION
ARG SPARK_VERSION
ARG SPARK_HOME
ARG TINI_VERSION

ENV ANALYTICS_ZOO_VERSION               ${ANALYTICS_ZOO_VERSION}
ENV SPARK_HOME                          ${SPARK_HOME}
ENV SPARK_VERSION                       ${SPARK_VERSION}
ENV ANALYTICS_ZOO_HOME                  /opt/analytics-zoo-${ANALYTICS_ZOO_VERSION}
ENV FLINK_HOME                          /opt/flink-${FLINK_VERSION}
ENV OMP_NUM_THREADS                     4
ENV NOTEBOOK_PORT                       12345
ENV NOTEBOOK_TOKEN                      1234qwer
ENV RUNTIME_SPARK_MASTER                local[4]
ENV RUNTIME_K8S_SERVICE_ACCOUNT         spark
ENV RUNTIME_K8S_SPARK_IMAGE             intelanalytics/hyper-zoo:${ANALYTICS_ZOO_VERSION}-${SPARK_VERSION}
ENV RUNTIME_DRIVER_HOST                 localhost
ENV RUNTIME_DRIVER_PORT                 54321
ENV RUNTIME_EXECUTOR_CORES              4
ENV RUNTIME_EXECUTOR_MEMORY             20g
ENV RUNTIME_EXECUTOR_INSTANCES          1
ENV RUNTIME_TOTAL_EXECUTOR_CORES        4
ENV RUNTIME_DRIVER_CORES                4
ENV RUNTIME_DRIVER_MEMORY               10g
ENV RUNTIME_PERSISTENT_VOLUME_CLAIM     myvolumeclaim
ENV SPARK_HOME                          /opt/spark
ENV HADOOP_CONF_DIR                     /opt/hadoop-conf
ENV BIGDL_VERSION                       ${BIGDL_VERSION}
ENV BIGDL_CLASSPATH                     ${ANALYTICS_ZOO_HOME}/lib/analytics-zoo-bigdl_${BIGDL_VERSION}-spark_${SPARK_VERSION}-${ANALYTICS_ZOO_VERSION}-jar-with-dependencies.jar
ENV JAVA_HOME                           /opt/jdk
ENV REDIS_HOME                          /opt/redis-5.0.5
ENV CS_HOME                             /opt/work/cluster-serving
ENV PYTHONPATH                          ${ANALYTICS_ZOO_HOME}/lib/analytics-zoo-bigdl_${BIGDL_VERSION}-spark_${SPARK_VERSION}-${ANALYTICS_ZOO_VERSION}-python-api.zip:${SPARK_HOME}/python/lib/pyspark.zip:${SPARK_HOME}/python/lib/py4j-*.zip:${CS_HOME}/serving-python.zip:/opt/models/research/slim
ENV PATH                                ${ANALYTICS_ZOO_HOME}/bin/cluster-serving:${JAVA_HOME}/bin:/root/miniconda3/bin:${PATH}
ENV TINI_VERSION                        ${TINI_VERSION}

RUN apt-get update --fix-missing && \
    apt-get install -y apt-utils vim curl nano wget unzip maven git && \
    apt-get install -y gcc g++ make && \
    apt-get install -y libsm6 libxext6 libxrender-dev && \
    rm /bin/sh && \
    ln -sv /bin/bash /bin/sh && \
    echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su && \
    chgrp root /etc/passwd && chmod ug+rw /etc/passwd

RUN mkdir -p /opt/analytics-zoo-examples/python && \
    mkdir -p /opt/analytics-zoo-examples/scala

COPY --from=analytics-zoo /opt/analytics-zoo-${ANALYTICS_ZOO_VERSION} /opt/analytics-zoo-${ANALYTICS_ZOO_VERSION}
COPY --from=analytics-zoo /opt/analytics-zoo-examples/python /opt/analytics-zoo-examples/python
COPY --from=spark /sbin/tini /sbin/tini
COPY --from=spark /opt/jdk /opt/jdk
COPY --from=spark /opt/spark /opt/spark
COPY --from=spark /opt/spark/kubernetes/dockerfiles/spark/entrypoint.sh /opt
COPY --from=redisenv /opt/redis-5.0.5 /opt/redis-5.0.5

#conda
RUN wget https://repo.anaconda.com/miniconda/Miniconda3-py37_4.8.3-Linux-x86_64.sh && \
    mkdir /root/.conda && \
    bash Miniconda3-py37_4.8.3-Linux-x86_64.sh -b && \
    rm -f Miniconda3-py37_4.8.3-Linux-x86_64.sh

#python
RUN apt-get install -y python3-minimal && \
    apt-get install -y build-essential python3 python3-setuptools python3-dev python3-pip && \
    pip3 install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir --upgrade setuptools && \
    pip install --no-cache-dir numpy scipy && \
    pip install --no-cache-dir pandas==1.0.3 && \
    pip install --no-cache-dir scikit-learn matplotlib seaborn jupyter jupyterlab wordcloud moviepy requests h5py opencv-python tensorflow==1.15.0 && \
    pip install --no-cache-dir torch==1.7.1 torchvision==0.8.2  && \
    ln -s /usr/bin/python3 /usr/bin/python && \
    #Fix tornado await process
    pip uninstall -y -q tornado && \
    pip install --no-cache-dir tornado && \
    python3 -m ipykernel.kernelspec && \
    pip install --no-cache-dir tensorboard && \
    pip install --no-cache-dir jep && \
    pip install --no-cache-dir cloudpickle && \
    pip install --no-cache-dir opencv-python && \
    pip install --no-cache-dir pyyaml && \
    pip install --no-cache-dir redis && \
    pip install --no-cache-dir keras && \
    pip install --no-cache-dir mxnet-mkl==1.6.0 && \
    pip install --no-cache-dir ray==1.2.0 && \
    pip install --no-cache-dir Pillow==6.1 && \
    pip install --no-cache-dir psutil aiohttp && \
    pip install --no-cache-dir py4j && \
    pip install --no-cache-dir gym gym[atari] && \
    pip install --no-cache-dir lz4 && \
    pip install --no-cache-dir ray[rllib] && \
    pip install --no-cache-dir cmake==3.16.3 && \
    pip install --no-cache-dir horovod==0.19.2 && \
    pip install --no-cache-dir tf_slim && \
    git clone https://github.com/tensorflow/models/ /opt/models

ADD ./start-notebook-spark.sh /opt
ADD ./start-notebook-k8s.sh /opt

RUN chmod a+x /opt/start-notebook-spark.sh && \
    chmod a+x /opt/start-notebook-k8s.sh && \
    chmod +x /sbin/tini

WORKDIR /opt/spark/work-dir

ENTRYPOINT [ "/opt/entrypoint.sh" ]


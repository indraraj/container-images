ARG DEBEZIUM_DOCKER_REGISTRY_PRIMARY_NAME
FROM $DEBEZIUM_DOCKER_REGISTRY_PRIMARY_NAME/base

LABEL maintainer="Debezium Community"

#
# Set the version, home directory, and MD5 hash.
# MD5 hash taken from http://kafka.apache.org/downloads.html for this version of Kafka
# These argument defaults can be overruled during build time but compatibility cannot be guaranteed when the defaults are not used.
#
ARG KAFKA_VERSION=3.9.0
ARG SCALA_VERSION=2.13
ARG SHA512HASH="5324C1F44D4C84EA469712C2CC3D2D15545C3716EDBB5353722DF9C661FCC78B031FCF07D1C4F0309C5FDB32686665DFB0CFFE55210CD3A1FE2A370538CB4E6D"

ENV KAFKA_VERSION=$KAFKA_VERSION \
    SCALA_VERSION=$SCALA_VERSION \
    KAFKA_HOME=/kafka \
    SHA512HASH=$SHA512HASH \
    KAFKA_URL_PATH=kafka/$KAFKA_VERSION/kafka_$SCALA_VERSION-$KAFKA_VERSION.tgz

ENV KAFKA_DATA=$KAFKA_HOME/data

#
# Create a user and home directory for Kafka
#
USER root
RUN groupadd -r kafka -g 1001 && useradd -u 1001 -r -g kafka -m -d $KAFKA_HOME -s /sbin/nologin -c "Kafka user" kafka && \
    chmod 755 $KAFKA_HOME
RUN mkdir $KAFKA_DATA && \
    mkdir $KAFKA_HOME/logs

#
# Change ownership and switch user
#
RUN chown -R kafka $KAFKA_HOME && \
    chgrp -R kafka $KAFKA_HOME

#
# 1.) Download Kafka, either from preferred host or archive
# 2.) Verify the contents and install, remove TGZ file
# 3.) Remove potentially exploitable classes (see CVE-2021-4104/DBZ-4447 CVE-2019-17571 CVE-2022-23302 CVE-2022-23305 CVE-2020-9493)
# 4.) Allow random UID to use Kafka (doing this for the bulk of files here, so as to avoid overhead of doing it in a separate layer)
RUN curl -fSL -o /tmp/kafka.tgz https://dlcdn.apache.org/$KAFKA_URL_PATH || curl -fSL -o /tmp/kafka.tgz https://archive.apache.org/dist/$KAFKA_URL_PATH &&\
    echo "$SHA512HASH /tmp/kafka.tgz" | sha512sum -c - &&\
    tar -xzf /tmp/kafka.tgz -C $KAFKA_HOME --strip-components 1 &&\
    rm -f /tmp/kafka.tgz &&\
    zip -d /kafka/libs/reload4j-1.2.25.jar org/apache/log4j/net/JMSAppender.class org/apache/log4j/net/SocketServer.class org/apache/log4j/net/JMSSink.class 'org/apache/log4j/jdbc/*' 'org/apache/log4j/chainsaw/*' &&\
    chmod -R g+w,o+w $KAFKA_HOME

COPY ./log4j.properties $KAFKA_HOME/config/log4j.properties
RUN chmod g+w,o+w $KAFKA_HOME/config/log4j.properties

# Back up config original files; they will be brought back in
# in docker-entrypoint.sh if no volume with user-provided config files is given
RUN mkdir $KAFKA_HOME/config.orig &&\
    mv $KAFKA_HOME/config/* $KAFKA_HOME/config.orig &&\
    chown -R kafka:kafka $KAFKA_HOME/config.orig

# Remove unnecessary files
RUN rm -f $KAFKA_HOME/libs/*-{sources,javadoc,scaladoc}.jar* &&\
    rm -r $KAFKA_HOME/site-docs

#
# The kafka-run-class.sh script generates the classpath for launching Kafka-related JVM, with entries
# containing the pattern "/bin/../libs", which fails to be resolved properly in some
# environments; the CLASSPATH is filled from "base_dir" environment variable that contains the relative
# path so it it is modified to contain absolute path using "realpath" command.
#
RUN sed -i 's/base_dir=\$(dirname \$0)\/../base_dir=\$(realpath \$(dirname \$0)\/..)/' $KAFKA_HOME/bin/kafka-run-class.sh

#
# Allow random UID to use Kafka
#
RUN chmod -R g+w,o+w $KAFKA_HOME

USER kafka

# Set the working directory to the Kafka home directory
WORKDIR $KAFKA_HOME

#
# Expose the ports and set up volumes for the data and logs directories
#
EXPOSE 9092
VOLUME ["/kafka/data","/kafka/logs","/kafka/config"]

COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["start"]

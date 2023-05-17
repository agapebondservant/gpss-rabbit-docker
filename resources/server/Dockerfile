FROM centos:8

ARG GPSS_VER_ARG=1.9.0
ENV GPSS_VER=${GPSS_VER_ARG}
ARG GPSS_BINARY_ARG=gpss-gpdb6-1.9.0-rhel8-x86_64.rpm
ENV GPSS_BINARY=${GPSS_BINARY_ARG}


COPY ${GPSS_BINARY} /tmp
COPY gpssrotate.conf /etc/logrotate.d/

RUN sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* && \
    sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* && \
    yum update -y && \
    yum install -y /tmp/${GPSS_BINARY} krb5-devel.x86_64

COPY gpss.json /usr/local/gpss-${GPSS_VER}

CMD ["sh", \
     "-c", \
     "echo 'Applying gpss_path.sh...'; source /usr/local/gpss-${GPSS_VER}/gpss_path.sh; echo 'Starting gpss...'; GPDB_CONNECT_TIMEOUT=30 gpss --config ${GPSS_HOME}/gpss.json --log-dir ${GPSS_HOME}/gpsslogs"]
FROM ubuntu:14.04
MAINTAINER  Arash Samadi (samadi@sub.uni-goettingen.de)
# 
# creating a predecessor for the LibreCat Image
#

# Add extra Repository for ElasticSearch
RUN apt-key adv --keyserver ha.pool.sks-keyservers.net --recv-keys 46095ACC8548582C1A2699A9D27D666CD88E42B4 \
    && echo "deb http://packages.elastic.co/elasticsearch/1.7/debian stable main" > /etc/apt/sources.list.d/elasticsearch-1.7.list
RUN sed -i '/multiverse/s/#[\ ]*deb/deb/g' /etc/apt/sources.list

ENV JAVA_HOME /usr/lib/jvm/java-7-openjdk-amd64
ENV PATH /usr/share/elasticsearch/bin:$PATH
ENV LIBRECATHOME /srv/LibreCat
ENV MYSQL_USER root
ENV MYSQL_PASSWORD alaki
ENV MYSQL_MAJOR 5.5
ENV MYSQL_VERSION 5.5.50
ENV DEBIAN_FRONTEND noninteractive
ENV ELASTIC_LOG /srv/LibreCat/logs/librecat_elasticsearch.log

# Prepare to install
RUN apt-get clean \
    && apt-get update \
    && apt-get upgrade -y \
    && apt-get dist-upgrade -y

RUN apt install -y build-essential
RUN apt install -y ca-certificates \
    wget \
    g++ \
    numactl \
    xz-utils \
    bsdmainutils \
    git \
    man-db \
    ImageMagick \
    libexpat1-dev \
    libssl-dev \
    libxml2-dev \
    libxslt1-dev \
    libgdbm-dev \
    libyaz-dev \
    libwrap0-dev \
    zlib1g \
    zlib1g-dev \
    binutils \
    pwgen \
    libaio1
RUN apt install -y perl-doc \
    cpanminus \
    libcapture-tiny-perl \
    libdbd-mysql-perl \
    libmodule-install-perl \
    libyaml-perl \
    libb-utils-perl \
    carton
RUN apt-get install -y openjdk-7-jre elasticsearch=1.7.5
RUN apt-get install -y gearman gearman-server gearman-tools

# MariaDB Server non-interactive
RUN groupadd -r mysql && useradd -r -g mysql mysql
#ENV debconf-set-selections 'mariadb-server-5.5 mysql-server/root_password password $MYSQL_PASSWORD'
#ENV debconf-set-selections 'mariadb-server-5.5 mysql-server/root_password_again password $MYSQL_PASSWORD'
# note: we're pulling the *.asc file from mysql.he.net instead of dev.mysql.com because the official mirror 404s that file for whatever reason - maybe it's at a different path?
RUN wget "http://dev.mysql.com/get/Downloads/MySQL-$MYSQL_MAJOR/mysql-$MYSQL_VERSION-linux2.6-x86_64.tar.gz" -O mysql.tar.gz \
    && wget "http://mysql.he.net/Downloads/MySQL-$MYSQL_MAJOR/mysql-$MYSQL_VERSION-linux2.6-x86_64.tar.gz.asc" -O mysql.tar.gz.asc \
    && export GNUPGHOME="$(mktemp -d)" \
# gpg: key 5072E1F5: public key "MySQL Release Engineering <mysql-build@oss.oracle.com>" imported
    && gpg --keyserver ha.pool.sks-keyservers.net --recv-keys A4A9406876FCBD3C456770C88C718D3B5072E1F5 \
    && gpg --batch --verify mysql.tar.gz.asc mysql.tar.gz \
    && rm -r "$GNUPGHOME" mysql.tar.gz.asc \
    && mkdir /usr/local/mysql \
    && tar -xzf mysql.tar.gz -C /usr/local/mysql --strip-components=1 \
    && rm mysql.tar.gz \
    && rm -rf /usr/local/mysql/mysql-test /usr/local/mysql/sql-bench \
    && rm -rf /usr/local/mysql/bin/*-debug /usr/local/mysql/bin/*_embedded \
    && find /usr/local/mysql -type f -name "*.a" -delete \
    && { find /usr/local/mysql -type f -executable -exec strip --strip-all '{}' + || true; } 

ENV PATH $PATH:/usr/local/mysql/bin:/usr/local/mysql/scripts

RUN mkdir -p /etc/mysql/conf.d \
    && { \
        echo '[mysqld]'; \
        echo 'skip-host-cache'; \
        echo 'skip-name-resolve'; \
        echo 'datadir = /var/lib/mysql'; \
        echo '!includedir /etc/mysql/conf.d/'; \
    } > /etc/mysql/my.cnf
RUN mkdir -p /var/lib/mysql /var/run/mysqld \
    && chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
    && chmod 777 /var/run/mysqld
RUN sed -i '/socket/ d' /etc/mysql/my.cnf \
    && sed -i '/\[mysqld\]/a bind-address\ =\ 0.0.0.0' /etc/mysql/my.cnf

# ElasticSearch
WORKDIR /usr/share/elasticsearch
RUN ln -s /etc/elasticsearch config
COPY conf/logging.yml /etc/elasticsearch/logging.yml

# git clone LibreCat
RUN cd /srv \
     && git clone --recursive https://github.com/subugoe/LibreCat.git \
     && cd LibreCat \
     && git fetch \
     && git checkout dev

WORKDIR $LIBRECATHOME
RUN mkdir logs \
    && touch $ELASTIC_LOG \
    && chown elasticsearch:elasticsearch $ELASTIC_LOG

#   Installing Perl-Modules and compiling everything
#   Finalizing setup and run Web-Interface under :5001
#
ENV LANG en_US.UTF-8
ENV LANGUAGE en.UTF-8
ENV LC_COLLATE en_US.UTF-8
ENV LC_CTYPE en_US.UTF-8
ENV LC_PAPER en_US.UTF-8
ENV LC_ADDRESS en_US.UTF-8
ENV LC_MONETARY en_US.UTF-8
ENV LC_NUMERIC en_US.UTF-8
ENV LC_TELEPHONE en_US.UTF-8
ENV LC_IDENTIFICATION en_US.UTF-8
ENV LC_MEASUREMENT en_US.UTF-8
ENV LC_TIME en_US.UTF-8
ENV LC_NAME en_US.UTF-8
RUN locale-gen en_US.UTF-8

# Building LibreCat
RUN sed -i 's/==/\>=/g' cpanfile
RUN sed -i '/Gearman::XS/ d' cpanfile
RUN cpanm -nq --installdeps Catmandu
RUN cpanm -nq --installdeps Catmandu::MARC
RUN cpanm -nq --installdeps .
RUN cpanm -nq --installdeps Gearman::XS Net::Telnet::Gearman
RUN carton install
RUN chmod +x bin/generate_forms.pl \
    && perl bin/generate_forms.pl

# Clean up
RUN apt-get clean \
    && apt-get purge -y binutils \
    wget \
    git \
    pwgen \
    binutils \
    && apt-get --purge autoremove -y \
    && rm -rf /var/lib/apt/lists/*

COPY conf/catmandu.local.yml .
COPY conf/catmandu.store.yml .
COPY gearman-entrypoint.sh ./gearboot.sh
COPY mysql-entrypoint.sh ./mysqlboot.sh
COPY docker-entrypoint.sh ./entrypoint.sh

VOLUME ["/usr/share/elasticsearch/data","/var/lib/mysql","/srv/LibreCat/public","/srv/LibreCat/view"]

EXPOSE 5001 3306 9200 9300 4730

CMD ["./entrypoint.sh"]

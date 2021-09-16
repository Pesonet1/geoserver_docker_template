FROM tomcat:8
# jdk11-openjdk-slim-buster 

ARG GWC_VERSION=1.18.4

ENV GWC_USER=geowebcache GWC_PASS=secured

COPY docker-entrypoint.sh /docker-entrypoint.sh

RUN apt-get update && apt-get install -y unzip wget

RUN cd /usr/local/tomcat/webapps/ \
    && wget --progress=bar:force:noscroll -c https://sourceforge.net/projects/geowebcache/files/geowebcache/${GWC_VERSION}/geowebcache-${GWC_VERSION}-war.zip/download -O geowebcache.war.zip \
    && unzip geowebcache.war.zip \
    && unzip geowebcache.war -d gwc \
    && rm -f geowebcache.war.zip \
    && rm -f geowebcache.war \
    && chmod +x /docker-entrypoint.sh

# Add custom CORS setup
COPY web.xml /usr/local/tomcat/conf/web.xml

# Set data_dir content for Geoserver container
COPY configuration /usr/local/tomcat/temp/geowebcache

# ADD ./gwc-azure-blob-1.18.4.jar /usr/local/tomcat/webapps/gwc/WEB-INF/lib

ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["catalina.sh", "run"]
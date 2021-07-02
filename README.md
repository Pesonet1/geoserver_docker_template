# Kartoza/geoserver docker image configuration/utilization

This repository is using [Kartoza Geoserver image](https://github.com/kartoza/docker-geoserver) for running Geoserver and managing Geoserver configuration. This repository leverage this image by adding configuration management & documentation for using it in future projects.

This repository assumes that Geoserver configuration (data_dir content) is managed inside version control and is only modified in developers local environment. This differs from the more regular approach where configuration is stored inside runtime environments fileshare/disk. Configuration is contained inside Docker image therefore configuration can be managed inside version control. However, some of the configuration is managed by mounting volumes into running container. These are GeoWebCache managed cached tiles & static data files.

## TODO

- Mount users file outside data_dir => credentials could be served as plain text?
  - `data_dir/security/usergroup/default/users.xml`
- Inject user passwords as plain text from env variables?
  - `<user enabled="true" name="admin" password="plain:${PASSWORD_FROM_ENV}"/>`
- Set all passwords to be plain and inject them from env variables?
- How to configure monitoring?

## Running

### Development

1. Set .env.prod file environment variables.

Enable UI by setting WEB_INTERFACE env variable to false. In addition, set EXISTING_DATA_DIR value to true to prevent overriding passwords

2. Run container locally with docker-compose:

`docker-compose -f ./docker-compose.yml up --build`

This runs container locally with mounted data_dir & data folders for managing configuration

3. Set Geoserver configuration via Geoserver UI

This will save configuration in data_dir folder

### Production

Build "production ready" image with following command

```
docker build . -f .\Dockerfile -t geopoc
```

#### Environment variables

Geoserver system setting -DALLOW_ENV_PARAMETRIZATION=true enables parametrizing some of the Geoserver settings via environment variables. These can be set with ${<ENV_VARIABLE>} template string. Note, that not all settings can be parametrizized.

```
AZURE_BLOB_CONTAINER=
AZURE_BLOB_ACCOUNT_NAME=
AZURE_BLOB_ACCESS_KEY=
```

In addition, Kartoza Geoserver image exposes following environemnt variables that can be set on runtime.

```
GEOSERVER_DATA_DIR=/opt/geoserver/data_dir
GEOWEBCACHE_CACHE_DIR=/opt/geoserver/data_dir/gwc

GEOSERVER_ADMIN_USER=admin
GEOSERVER_ADMIN_PASSWORD=admin_salasana

If EXISTING_DATA_DIR is set to true update_passwords.sh script is run (if .updatepassword.lock file is not set in data_dir)

EXISTING_DATA_DIR=true

Add desired community extensions

COMMUNITY_EXTENSIONS=gwc-azure-blobstore-plugin

Disabled admin UI if set to true

WEB_INTERFACE=true

See optimization options here https://docs.geoserver.org/stable/en/user/production/container.html

CATALINA_OPTS=-XX:SoftRefLRUPolicyMSPerMB=45000 -XX:+UseG1GC -server -Xbootclasspath/a:$MARLIN_JAR -Dsun.java2d.renderer=org.marlin.pisces.MarlinRenderingEngine -DALLOW_ENV_PARAMETRIZATION=true
INITIAL_MEMORY=2G # Adjust
MAXIMUM_MEMORY=2G # Adjust

Disable tomcat UI if set to false

TOMCAT_EXTRAS=false
TOMCAT_USER=
TOMCAT_PASS=

Sets let's encrypt certification if true and own certs not provided via volume -v /etc/certs:/etc/certs

SSL=true
PKCS12_PASSWORD=
JKS_KEY_PASSWORD=
JKS_STORE_PASSWORD=
```

#### Mount data directory

Mount following directories as volumes

`data => /opt/geoserver/data_dir/data`

#### Mount JNDI configuration via tomcat

Optionally mount JNDI configuration for database connection configuration

`./tomcat/context.xml => /usr/local/tomcat/conf/context.xml`

Create JNDI connection with name i.e. java:comp/env/jdbc/postgres with following example context.xml file

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>WEB-INF/tomcat-web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>

    <Resource
      name="jdbc/postgres"
      auth="Container"
      type="javax.sql.DataSource"
      driverClassName="org.postgresql.Driver"
      url="jdbc:postgresql://localhost:5432/test_db"
      username="postgres" password="postgres"
      maxActive="20"
      initialSize="0"
      minIdle="0"
      maxIdle="8"
      maxWait="10000"
      timeBetweenEvictionRunsMillis="30000"
      minEvictableIdleTimeMillis="60000"
      testWhileIdle="true"
      validationQuery="SELECT 1"
    />
</Context>
```

#### Azure (container registry)

Following commands enable pushing container to acr registry

```
az login --tenant <tenant-id>
az acr login --name <container-registry>
docker login <container-registry>.azurecr.io
docker tag geopoc <container-registry>.azurecr.io/geopoc
docker push <container-registry>.azurecr.io/geopoc
```

## Configuration

Following chapters contain some information on how to configure some of the image settings

### Plugins

Add own plugins inside Dockerfile configuration

`/usr/local/tomcat/webapps/geoserver/WEB-INF/lib`

### Static data

Static data are added to running container by mounting data dir as volume in `/opt/geoserver/data_dir/data` folder.

If locally some files are not available for datastore setting, use some mock data sources and change data dir location manually from datastore configuration. Note, that is won't obviously work locally but only in running environment

### Cache (Azure BlobStore)

This repository contains configuration for using Azure Blob Storage as cached tile data source. Following environment variables are needed:

```
AZURE_BLOB_CONTAINER=
AZURE_BLOB_ACCOUNT_NAME=
AZURE_BLOB_ACCESS_KEY=
```

#### GWC REST API (Seeding)

GWC REST API can be used for triggering seeding processes.

Get GWC layers

```
GET http://localhost:8080/geoserver/gwc/rest/layers/
```

Post seed request

```
POST http://localhost:8080/geoserver/gwc/rest/seed/poc:mml_milj_2020.json

BODY {
  "seedRequest": {
    "name": "poc:mml_milj_2020",
    "srs": {
      "number": 3067
    },
    "zoomStart": 0,
    "zoomStop": 5,
    "format": "image/png",
    "type": "seed",
    "gridSetId": "JHS180",
    "threadCount": 4
  }
}
```

Get seed status

```
GET http://localhost:8080/geoserver/gwc/rest/seed/poc:mml_milj_2020.json

RESPONSE {
  "long-array-array": [
    [
      4592, # tiles completed
      5645, # tile estimate count
      119, # time remaining
      1, # task id
      1 # task status (-1 = ABORTED, 0 = PENDING, 1 = RUNNING, 2 = DONE)
    ]
  ]
}
```

Kill all seeding processes or only for one layer

```
POST http://localhost:8080/geoserver/gwc/rest/seed.json?kill_all=all
POST http://localhost:8080/geoserver/gwc/rest/seed/poc:mml_milj_2020.json?kill_all=all
```

## Credentials management

Geoserver has a master password that is used for encrypting all secrets inside Geoserver configurations. Additionally, Geoserver has admin user that is used for logging in web ui & managing Geoserver configuration. Latter is set during initial data_dir setup by following env variables:

```
GEOSERVER_ADMIN_USER=admin
GEOSERVER_ADMIN_PASSWORD=geoserver
```

By default master password is geoserver. This should be changed after initializing data_dir via ui.

Master password can be fetched by requesting `<geoserver-url>/geoserver/rest/security/masterpw.xml` url with admin user credentials (basic auth). See https://github.com/kartoza/docker-geoserver/issues/69 for more information how to change it via REST API call.

**NOTE!** Credentials should only be updated and set on local environment via UI. On server environment UI should be disabled.
**NOTE!** Master and admin passwords should be kept in KeyVault or similar in order to prevent locking Geoserver.
**NOTE!** Since master password is used for encrypting data_dir secrets etc. it doesn't matter if they are included in version control.

After initialization data_dir contains encrypted master password and users (i.e. admin) and accessing running Geoserver it is only possible with the set admin password. These can be changed on locally running Geoserver web ui.

Current image credentials

Master password => master_salasana
Admin user password => admin_salasana

Passwords can be injected inside xml configuration with `${<ENVIRONMENT_VARIABLE>}`

https://docs.geoserver.org/stable/en/user/community/backuprestore/configtemplate.html
https://docs.geoserver.org/master/en/user/data/app-schema/property-interpolation.html

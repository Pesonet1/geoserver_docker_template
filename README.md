# Geoserver container configuration

This repository is using [Kartoza Geoserver image](https://github.com/kartoza/docker-geoserver) for running Geoserver and managing Geoserver configuration. This repository leverage this image by adding configuration management & documentation for using it in future projects.

This repository assumes that Geoserver configuration (data_dir content) is managed inside version control and is only modified in developers local environment. This differs from the more regular approach where configuration is stored inside runtime environments fileshare/disk. Configuration is contained inside Docker image therefore configuration can be managed inside version control. However, some of the configuration is managed by mounting volumes into running container. These are GeoWebCache managed cached tiles & static data files.

## TODO

Mount users file outside data_dir => credentials could be served as plain text?
- data_dir/security/usergroup/default/users.xml

Inject user passwords as plain text from env variables?
<user enabled="true" name="admin" password="plain:${PASSWORD_FROM_ENV}"/>

Set all passwords to be plain and inject them from env variables?

## Running

### Development

1. Set .env.prod file environment variables.

Enable UI by setting WEB_INTERFACE env variable to false. In addition, set EXISTING_DATA_DIR value to true to prevent overriding passwords

2. Run container locally with docker-compose:

`docker-compose -f ./docker-compose.yml up --build`

This runs container locally with mounted data_dir & data folders for managing configuration

3. Set Geoserver configuration via Geoserver UI

### Production

Build "production ready" image with following command

```
docker build . -f .\Dockerfile -t geopoc
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

#### Env variables

Geoserver system setting -DALLOW_ENV_PARAMETRIZATION=true enables parametrizing some of the Geoserver settings via environment variables. These can be set with ${<ENV_VARIABLE>} template string. Note, that not all settings can be parametrizized.

```
AZURE_BLOB_CONTAINER=
AZURE_BLOB_ACCOUNT_NAME=
AZURE_BLOB_ACCESS_KEY=
```

In addition, Kartoza Geoserver image exposes following env variables to be set on runtime.

```
GEOSERVER_DATA_DIR=/opt/geoserver/data_dir
GEOWEBCACHE_CACHE_DIR=/opt/geoserver/data_dir/gwc

GEOSERVER_ADMIN_USER=admin
GEOSERVER_ADMIN_PASSWORD=admin_salasana

If EXISTING_DATA_DIR is set to true update_passwords.sh script is run

EXISTING_DATA_DIR=true

Add desired community extensions

COMMUNITY_EXTENSIONS=gwc-azure-blobstore-plugin
WEB_INTERFACE=true

See optimization options here https://docs.geoserver.org/stable/en/user/production/container.html

CATALINA_OPTS=-XX:SoftRefLRUPolicyMSPerMB=45000 -XX:+UseG1GC -server -Xbootclasspath/a:$MARLIN_JAR -Dsun.java2d.renderer=org.marlin.pisces.MarlinRenderingEngine -DALLOW_ENV_PARAMETRIZATION=true
INITIAL_MEMORY=2G # Adjust
MAXIMUM_MEMORY=2G # Adjust

TOMCAT_EXTRAS=false
TOMCAT_USER=
TOMCAT_PASS=

SSL=true
PKCS12_PASSWORD=
JKS_KEY_PASSWORD=
JKS_STORE_PASSWORD=
```

Mount following directories as volumes

data => /opt/geoserver/data_dir/data

Optionally mount JNDI configuration

./tomcat/context.xml => /usr/local/tomcat/conf/context.xml

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

## Development workflow

1. Locally manage Geoserver
2. Build and push Docker image
3. Apply new image on runtime environment

blob store configuration
static data configuration (how to add locally a file that is not present locally)
  - first create with mock file and modify path to file (mounted volume)
geoserver data_dir configuration
credential management
how to add new datastore / layer / cache
how to change password (master, admin)
custom vs kartoza geoserver image
how to set monitoring
how to set logging
how to scale out geoserver
  - app service & clustering
  - 		- https://docs.geoserver.org/stable/en/user/community/jms-cluster/index.html
		- https://docs.geoserver.geo-solutions.it/edu/en/clustering/clustering/active/index.html

## Configuration

https://geopoc.azurewebsites.net/geoserver/gwc/home

https://geopoc.azurewebsites.net/geoserver/gwc/rest/reload

Add own plugins inside Dockerfile configuration

/usr/local/tomcat/webapps/geoserver/WEB-INF/lib


### Cache (Azure BlobStore)

TODO

Evaluate costs and size call amounts

Standalone GWC instead of embedded one?
  - how to configure
  - add example with docker compose?

### Updating new image

## Credentials management

Geoserver has a master password that is used for encrypting all secrets inside Geoserver configurations. Additionally, Geoserver has admin user that is used for logging & managing Geoserver configuration. Latter is set during initial data_dir setup by following env variables:

GEOSERVER_ADMIN_USER=admin
GEOSERVER_ADMIN_PASSWORD=geoserver

By default master password is geoserver. This should be changed after initializing data_dir via ui.

Master password can be fetched by requesting `<geoserver-url>/geoserver/rest/security/masterpw.xml` url with admin user credentials (basic auth). See https://github.com/kartoza/docker-geoserver/issues/69 for more information how to change it via REST API call.

NOTE! Credentials should only be updated and set on local environment via UI. On server environment UI should be disabled.
NOTE! Master and admin passwords should be kept in KeyVault or similar in order to prevent locking Geoserver.
NOTE! Since master password is used for encrypting data_dir secrets etc. it doesn't matter if they are included in version control.

After initialization data_dir contains encrypted master password and users (i.e. admin) and accessing running Geoserver it is only possible with the set admin password

Current image credentials

Master password => master_salasana
Admin user password => admin_salasana

Passwords can be injected inside xml configuration with ${<ENVIRONMENT_VARIABLE>}

https://docs.geoserver.org/stable/en/user/community/backuprestore/configtemplate.html
https://docs.geoserver.org/master/en/user/data/app-schema/property-interpolation.html
#Json JDBC based map store for Hazelcast

Can be used to persist a Hazelcast <String, Serializable> map into a SQL database. Serializing the objects to Json.
The intended use is to store only simple serializable objects (think of Strings, Integers and small shallow classes).

**Please note that this implementation isn't optimized for large maps.**

### Table

The table name for a map is based on the map name itself.
It can be prefixed (e.g. with 'hz_'), for this see: [Configuration](#Configuration)

The table should already exist (you have to create it yourself).
For example in MySQL you can create a table like this:

```sql
CREATE TABLE name_of_your_map (
  key_md5 VARCHAR(32) PRIMARY KEY,
  key_org VARCHAR(256) NOT NULL,
  classname VARCHAR(256) NOT NULL,
  value TEXT NULL,
  lastUpdated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
)
```

_The 'lastUpdated' column is optional and not required_

### Configuration <a name="Configuration"></a>

There are a couple of things that can be configured.
Below you find the settings and how a value can be assigned.

Setting            | Map property    | System property                  | Environment variable
------------------ | --------------- | -------------------------------- | --------------------------------
DB driver class    | dbDriver        | hz.mapstore.jdbc.dbDriver        | HZ_MAPSTORE_JDBC_DBDRIVER
DB URL             | dbUrl           | hz.mapstore.jdbc.dbUrl           | HZ_MAPSTORE_JDBC_DBURL
DB username        | dbUser          | hz.mapstore.jdbc.dbUser          | HZ_MAPSTORE_JDBC_DBUSER
DB password        | dbPass          | hz.mapstore.jdbc.dbPass          | HZ_MAPSTORE_JDBC_DBPASS
Table name prefix  | tableNamePrefix | hz.mapstore.jdbc.tableNamePrefix | HZ_MAPSTORE_JDBC_TABLENAMEPREFIX
Output pretty Json | jsonPretty      | hz.mapstore.jdbc.jsonPretty      | HZ_MAPSTORE_JDBC_JSONPRETTY

Environment variables will be used in favor of system properties, which in turn will be used in favor of values the map properties.
Meaning that settings will be loaded in this order (first found will be used): Environment variable > System property > Map property

_When you provide a JDBC 4 compatible database driver on the class path it should not be necessary to use the DB driver class setting._

### Enabling the map store

This can be done in Hazelcast's XML configuration like this:

```xml
<map name="name_of_your_map">
  <map-store enabled="true">
    <factory-class-name>nl.brightbits.hazelcast.JsonJdbcMapStoreFactory</factory-class-name>
    <properties>
      <property name="jsonPretty">true</property>
    </properties>
  </map-store>
</map>
```

If you enable persistence for multiple maps I would advice to configure the database URL etc. via system properties
instead of duplicating this for each map in the XML.

### Docker

When running Hazelcast server in Docker you can use this map store by creating a custom Docker file like this:

```
FROM hazelcast/hazelcast:3.6.2

# Add Hazelcast configuration to image
ADD hazelcast.xml $HZ_HOME

# Add mapstore to image
ADD https://github.com/rlindooren/hazelcast-json-jdbc-mapstore/releases/download/1.0/hazelcast-json-jdbc-mapstore-1.0.jar $HZ_HOME/hazelcast-json-jdbc-mapstore.jar

# Add required dependencies by mapstore to image
ADD http://central.maven.org/maven2/com/google/code/gson/gson/2.6.2/gson-2.6.2.jar $HZ_HOME/gson.jar
ADD http://central.maven.org/maven2/org/apache/commons/commons-lang3/3.4/commons-lang3-3.4.jar $HZ_HOME/commons-lang3.jar
ADD http://central.maven.org/maven2/commons-codec/commons-codec/1.10/commons-codec-1.10.jar $HZ_HOME/commons-codec.jar
ADD http://central.maven.org/maven2/org/slf4j/slf4j-api/1.6.4/slf4j-api-1.6.4.jar $HZ_HOME/slf4j-api.jar

# Add applicable database driver (use your own driver here)
ADD http://central.maven.org/maven2/org/postgresql/postgresql/9.4.1208.jre7/postgresql-9.4.1208.jre7.jar $HZ_HOME/postgresql.jar

ENV CLASSPATH \
$HZ_HOME/hazelcast-json-jdbc-mapstore.jar:\
$HZ_HOME/postgresql.jar:\
$HZ_HOME/gson.jar:\
$HZ_HOME/commons-lang3.jar:\
$HZ_HOME/commons-codec.jar:\
$HZ_HOME/slf4j-api.jar:

ENV JAVA_OPTS \
-Dhz.mapstore.jdbc.dbUrl=jdbc:postgresql://your_host/your_db \
-Dhz.mapstore.jdbc.dbUser=your_username \
-Dhz.mapstore.jdbc.dbPass=your_password
```

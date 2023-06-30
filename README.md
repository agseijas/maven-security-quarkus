# Issue with quarkus and maven and secured proxied repositories

To setup this we need a maven repository proxied to maven central which requires authenticated 
access to it.

This was tested on:
- Ubuntu 22.04
- Java 20
- Maven 3.9.3

## 1. Run nexus container
To do this you might just use nexus oss container:

`docker run -d -p 8083:8081 --name nexus sonatype/nexus:oss`

Then log in (admin/admin123) and disable anonymous access:
Administration -> Server -> Disable Anonymous Access

## 2. Setup maven settings to use repo with auth credentials

Create a master password to create a `~/.m2/settings-security.xml`: https://maven.apache.org/guides/mini/guide-encryption.html#how-to-create-a-master-password

Then get the encrypted password:

`mvn --encrypt-password admin123`

And then create the settings.xml file to let maven use the proxy repo:

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>mynexusoss</id>
      <username>admin</username>
      <password>{COQLCE6DU6GtcS5P=}</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>mynexusoss</id>
      <mirrorOf>*</mirrorOf>
       <url>http://localhost:8083/nexus/content/repositories/central</url>
    </mirror>
  </mirrors>
  <profiles>
    <profile>
      <id>hide-central-which-is-redirected-to-nexus</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>http://bogus-central</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
     <pluginRepositories>
        <pluginRepository>
          <id>central</id>
          <url>http://bogus-central</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>hide-central-which-is-redirected-to-nexus</activeProfile>
  </activeProfiles>
</settings>
```

## 3. Clean up repo

Make sure you clean your maven repo (so that the quarkus dependencies are pulled again). For example:

`rm -rf ~/.m2/repository/io/quarkus`

(NOTE: with version 3.1.3.Final it's enough if you delete the `io.quarkus:quarkus-
resteasy-reactive-kotlin` local cache)

## 4. Run the scenario

Run:

`mvn clean install`

And then it'll error during the `GreetingResourceTest` test run with the 401 error:

```text
Caused by: org.apache.http.client.HttpResponseException: status code: 401, reason phrase: Unauthorized (401)
        at org.eclipse.aether.transport.http.HttpTransporter.handleStatus(HttpTransporter.java:527)
        at org.eclipse.aether.transport.http.HttpTransporter.execute(HttpTransporter.java:396)
        at org.eclipse.aether.transport.http.HttpTransporter.implGet(HttpTransporter.java:343)
        at org.eclipse.aether.spi.connector.transport.AbstractTransporter.get(AbstractTransporter.java:64)
        at org.eclipse.aether.connector.basic.BasicRepositoryConnector$GetTaskRunner.runTask(BasicRepositoryConnector.java:482)
        at org.eclipse.aether.connector.basic.BasicRepositoryConnector$TaskRunner.run(BasicRepositoryConnector.java:414)
        ... 67 more
```

To make it pass and fix it until, for example, a new dependency appears during test phase on a 
newer/older quarkus version upgrade/downgrade; you'll have to provide the settings security parameter to maven:

`mvn -Dsettings.security=~/.m2/settings-security.xml clean install`


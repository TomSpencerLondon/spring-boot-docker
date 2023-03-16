#Spring Boot and Docker

### Add fabric8 maven plugin

```xml
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.20.0</version>

                <configuration>
                    <dockerHost>http://127.0.0.1:2375</dockerHost>
<!--                    <dockerHost>unix:///var/run/docker.sock</dockerHost>-->

                    <verbose>true</verbose>
                    <images>
                        <image>
                            <name>${docker.image.prefix}/${docker.image.name}</name>
                            <build>
                                <dockerFileDir>${project.basedir}/src/main/docker/</dockerFileDir>

                                <!--copies artficact to docker build dir in target-->
                                <assembly>
                                    <descriptorRef>artifact</descriptorRef>
                                </assembly>
                                <tags>
                                    <tag>latest</tag>
                                    <tag>${project.version}</tag>
                                </tags>
                            </build>
                        </image>
                    </images>
                </configuration>
            </plugin>
```

Here we add the fabric8 plugin:
```xml
<dockerHost>unix:///var/run/docker.sock</dockerHost>
```

This tells the plugin where you dockerfile is kept:
```xml
<dockerFileDir>${project.basedir}/src/main/docker/</dockerFileDir>
```

The following copies build artifacts into the target directory:
```xml
<assembly>
    <descriptorRef>artifact</descriptorRef>
</assembly>
```
The fabric8 plugin copies the artifacts into its own directory. 

This adds two tags:
```xml
<tags>
    <tag>latest</tag>
    <tag>${project.version}</tag>
</tags>
```
The latest tag and the project version.

### Using the fabric8 plugin
We run the maven clean package first and then docker build.
```bash
mvn clean package
mvn clean package docker:build
```
The ```docker:build``` command is the same as:
```bash
docker build -t spring-boot-docker .
```

To push the image to docker we can use:
```java
mvn clean package docker:build docker:push
```
We also need to update settings.xml inside .m2 folder:
```xml

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>docker.io</id>
            <username></username>
            <password></password>
        </server>
    </servers>
</settings>
```
This allows us to push the image to dockerhub (hub.docker.com).

![image](https://user-images.githubusercontent.com/27693622/225072954-aed7fd84-8e5a-4e7f-9ffb-36d778981ade.png)

### Add dynamic versioning
```bash

FROM openjdk

ARG artifactId
VOLUME /tmp
ADD maven/spring-boot-docker-${artifactId}.jar myapp.jar
RUN sh -c "touch /myapp.jar"
ENTRYPOINT ["java", "-Djava.security.eqd=file:/dev/./urandom", "-jar", "/myapp.jar"]
```
The artifactId is defined in the pom.xml file:
```xml
<configuration>
    <buildArgs>
        <artifactId>${project.version}</artifactId>
    </buildArgs>
</configuration>
```

We can now see different versions of the docker image as we push to dockerhub:
```bash
mvn clean package docker:build docker:push
```

![image](https://user-images.githubusercontent.com/27693622/225078310-bfc73a68-6318-4862-bd71-a099705d77e0.png)

### Adding port mapping

We can specify ports with the fabric8 plugin with the run configuration:

```xml
<run>
    <ports>
        <port>8081:8080</port>
    </ports>
</run>
```
You can then run the application with:
```bash
mvn docker:run
```
Run without interactive:
```bash
 mvn docker:start
```
We can also do:
```bash
mvn docker:stop
```

#### Running page view microservice with tutorial site:

```bash
docker network create pageview

docker run --name mysqldb -p 3306:3306  --network pageview -e MYSQL_DATABASE=pageviewservice -e MYSQL_ALLOW_EMPTY_PASSWORD=yes -d mysql

docker run -d --name rabbitmq -p 8081:15672 -p 5671:5671 -p 5672:5672  --network pageview rabbitmq:3-management

docker run --name pageviewservice -p 8082:8081 \
 --network pageview \
-e SPRING_DATASOURCE_URL=jdbc:mysql://mysqldb:3306/pageviewservice \
-e SPRING_PROFILES_ACTIVE=mysql  \
-e SPRING_RABBITMQ_HOST=rabbitmq \
springframeworkguru/pageviewservice
```

### JAXB with newer java versions
- This is needed for this dependency:
```xml
<dependency>
    <groupId>guru.springframework</groupId>
    <artifactId>page-view-client</artifactId>
    <version>0.0.2</version>
</dependency>

```
Add glassfish dependency for java:
```xml
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.1</version>
</dependency>
```

### RabbitMQ message for page visit:
```bash
Sending Message
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<pageViewEvent>
    <correlationId>542b8cf6-61cd-41e0-940d-e71c5f10fc34</correlationId>
    <pageUrl>springfarmework.guru/product/1</pageUrl>
    <pageViewDate>2023-03-15T12:11:48.396Z</pageViewDate>
</pageViewEvent>
```
This is then received by the page view service:
```bash
Got Message!
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<pageViewEvent>
    <correlationId>542b8cf6-61cd-41e0-940d-e71c5f10fc34</correlationId>
    <pageUrl>springfarmework.guru/product/1</pageUrl>
    <pageViewDate>2023-03-15T12:11:48.396Z</pageViewDate>
</pageViewEvent>
```

### Run
Build with:
```bash
mvn clean package
mvn clean package docker:build
```
Run with:
```bash
mvn docker:run
```

### Add network for connecting to RabbitMQ
Add custom network config for Fabric8 with RabbitMQ:
```bash
<run>
    <ports>
        <port>8090:8080</port>
    </ports>
    <network>
        <mode>custom</mode>
        <name>pageview</name>
    </network>
</run>
```

### Deployment to docker
We set up integration tests with the maven surefire plugin:
```bash

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.0.0</version>
    <executions>
        <execution>
            <id>integration-test</id>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>

    </executions>
</plugin>
```
This runs Tests with the suffix IT automatically:
```bash
public class FakeIntegrationTestsIT {

    @Test
    public void someFakeTestOne() throws Exception {
        System.out.println("Test 1 .");

        Thread.sleep(100);

        System.out.println("Test 1 . .");

        Thread.sleep(100);

        System.out.println("Test 1 . .");

        Thread.sleep(100);

        System.out.println("Test 1 . . .");

        Thread.sleep(100);

        System.out.println("Test 1 . . . .");

        Thread.sleep(100);

        System.out.println("Test 1 . . . . .");
    }
}
```

You can also set the execution in the fabric8 plugin to check the builds the fabric8 docker images:
```bash
<executions>
    <execution>
        <id>start</id>
        <phase>pre-integration-test</phase>
        <goals>
            <goal>build</goal>
            <goal>start</goal>
        </goals>
    </execution>
    <execution>
        <id>stop</id>
        <phase>post-integration-test</phase>
        <goals>
            <goal>stop</goal>
        </goals>
    </execution>
</executions>
```

### Example mvn command for Continuous Integration
```bash
mvn clean package verify docker:push
```

### Docker Compose
- use yml to work with docker compose

```yaml
# This is a comment
somename: value # this is also a comment
anothername: '#value' # that is a value

# spring.datasource.initial-size=1
# spring.datasource.max-active=10
# spring.datasource.min-idle=1
# spring.datasource.max-idle=5

spring:
  datasource:
    initial-size: 1
    max-active: 10
    min-idle: 1
    max-idle: 5
    
somelist:
  - value1: one
  - value2: two
# list of maps
anothermaplist:
  - keyvalue: 1
    value: 'my value'
  - keyvalue: 2
    value: 'my value'
```
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

### Wordpress example with docker compose

```bash
version: '2'

services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: wordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    ports:
      - 3306:3306

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - 8000:80
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
volumes:
  db_data:

```
Here the database is started first as specified by ```depends_on: -db```

### Docker-compose

```bash
tom@tom-ubuntu:~/Projects/spring-boot-docker/src/main/scripts/wordpress$ docker-compose up -d
Creating network "wordpress_default" with the default driver
Creating volume "wordpress_db_data" with default driver
Pulling db (mysql:5.7)...
5.7: Pulling from library/mysql
7b659169cb92: Downloading [====================>                              ]  20.93MB/50.47MB
e47c3f06a3f5: Download complete
e0656a27f56e: Download complete
b9f33f34ef42: Download complete
9fc5cbaf8704: Download complete
5fb8407bef93: Download complete
f8be4a2d031b: Downloading [=====================>                             ]  11.04MB/25.52MB
b6cb2bff25e3: Download complete
056cc4a5c89a: Download complete
842896973144: Download complete
e55b6e95b292: Waiting
```

### Check running docker containers:
```bash
tom@tom-ubuntu:~$ docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                                   NAMES
d12d44b15455   wordpress:latest   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds   0.0.0.0:8000->80/tcp, :::8000->80/tcp   wordpress_wordpress_1
06032a86b97a   mysql:5.7          "docker-entrypoint.s…"   6 seconds ago   Up 3 seconds   3306/tcp, 33060/tcp                     wordpress_db_1
```

go to port 8080:
![image](https://user-images.githubusercontent.com/27693622/225575424-32514f39-07bc-4b29-8710-e725c04b54ca.png)

### Shut down
```bash
docker-compose down
```

#### Clear volumes
```bash
docker volume rm $(docker volume ls -f dangling=true -q)
```

### Spring docker-compose

```yaml
version: '3'

services:
  pageviewservice:
    image: tomspencerlondon/pageviewservice
    ports:
      - "8082:8081"
    networks:
      - newrabbit
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health"]
      interval: 1m30s
      timeout: 10s
      retries: 3
networks:
  newrabbit:
    external: true
```

Running spring-boot-docker with ```mvn docker:run``` and then we see logs in the pageviewservice:
```bash


pageviewservice_1  | Got Message!
pageviewservice_1  | <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
pageviewservice_1  | <pageViewEvent>
pageviewservice_1  |     <correlationId>feb10afb-5330-4a35-a2ba-4c303833ffe5</correlationId>
pageviewservice_1  |     <pageUrl>springfarmework.guru/product/1</pageUrl>
pageviewservice_1  |     <pageViewDate>2023-03-16T11:32:01.415Z</pageViewDate>
pageviewservice_1  | </pageViewEvent>
pageviewservice_1  | 

```

### Docker compose again

```bash
version: '3.5'

services:
  pageviewservice:
    depends_on:
      - mysqldb
      - rabbitmq
    image: tomspencerlondon/pageviewservice
    ports:
      - "8082:8081"
    networks:
      - newrabbit
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health"]
      interval: 1m30s
      timeout: 10s
      retries: 3
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "8081:15672"
      - "5671:5671"
      - "5672:5672"
    networks:
      - newrabbit
  mysqldb:
    image: mysql
    ports:
      - "3306:3306"
    networks:
      - newrabbit
    restart: always
    environment:
      MYSQL_DATABASE: pageviewservice
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
  springbootdocker:
    depends_on:
      - rabbitmq
    image: tomspencerlondon/springbootdocker
    ports:
      - "8090:8080"
    networks:
      - newrabbit
networks:
  newrabbit:
    name: newrabbit
    external: false
```

Above we have multiple services within docker-compose. We are using images that we have pushed to dockerhub.

### Example docker config
Dockerfile:
```bash
FROM openjdk:17-oracle

EXPOSE 8080

COPY target/example-project-0.0.1-SNAPSHOT.jar example-0.0.1-SNAPSHOT.jar

ENTRYPOINT ["java","-jar","example-0.0.1-SNAPSHOT.jar"]
```


Docker Build, Tag & Push
Step 1: Build Image
docker build -t web_app1 .
Step 2: Tag Your docker image
docker tag web_app1:latest gcr.io/<GCP_Project_ID>/web_app1:latest
Step 3: Verify image is created and you can see the new tag
docker image list
Step 4: One time setup
gcloud auth configure-docker
gcloud components update
Step 5: Push your docker image
Step 6 : Verify that you can pull image docker run -ti --rm -p 8080:8080 \ website
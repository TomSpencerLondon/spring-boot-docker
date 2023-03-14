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
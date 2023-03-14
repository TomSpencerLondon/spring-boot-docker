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
                                <dockerFileDir>${project.basedir}/target/dockerfile/</dockerFileDir>

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

Here we add the fabric8 plugin. I am using localhost because I am running on linux - for unix we use the commented line:
```xml
<dockerHost>unix:///var/run/docker.sock</dockerHost>
```

This tells the plugin where you dockerfile is kept:
```xml
<dockerFileDir>${project.basedir}/target/dockerfile/</dockerFileDir>
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


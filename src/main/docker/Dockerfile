FROM openjdk:11

ARG version
VOLUME /tmp
ADD maven/spring-boot-docker-${version}.jar myapp.jar
RUN sh -c "touch /myapp.jar"
ENTRYPOINT ["java", "-Djava.security.eqd=file:/dev/./urandom", "-jar", "/myapp.jar"]

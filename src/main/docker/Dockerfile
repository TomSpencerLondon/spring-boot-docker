FROM openjdk

ARG artifactId
VOLUME /tmp
ADD maven/spring-boot-docker-${artifactId}.jar myapp.jar
RUN sh -c "touch /myapp.jar"
ENTRYPOINT ["java", "-Djava.security.eqd=file:/dev/./urandom", "-jar", "/myapp.jar"]

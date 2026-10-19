FROM registry.access.redhat.com/hi/openjdk:25.0.4-builder AS builder
WORKDIR /opt/app

COPY . .

RUN ./mvnw clean package -DskipTests

FROM registry.access.redhat.com/hi/openjdk:25.0.4-runtime
WORKDIR /opt/app

COPY --from=builder /opt/app/target/spring-petclinic-*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

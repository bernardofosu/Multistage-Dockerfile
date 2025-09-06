A. Improved multi-stage (recommended)
# --- Stage 1: Build (Maven + JDK) ---
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /usr/src/app

# Cache deps
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline

# Build app
COPY src ./src
RUN mvn -q -DskipTests package

# --- Stage 2: Runtime (Tomcat only) ---
FROM tomcat:9.0-jdk17-temurin
WORKDIR /usr/local/tomcat/webapps

# Deploy as ROOT.war
COPY --from=build /usr/src/app/target/java-tomcat-maven-example.war ./ROOT.war

# (Optional) health: Tomcat listens on 8080
EXPOSE 8080


Why this rocks: tiny final image, only Tomcat + WAR, fast repeat builds.

B. If you insist on the Ubuntu style (fixed)

If you really need a hand-rolled image, at least keep Tomcat in the foreground and drop the risky bits:

FROM ubuntu:22.04

RUN apt-get update -y && \
    apt-get install -y curl ca-certificates default-jdk maven && \
    rm -rf /var/lib/apt/lists/*

# Build (not ideal for prod, but OK if you must)
WORKDIR /src
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

# Install Tomcat (pin a maintained version!)
ENV TOMCAT_VER=9.0.89
WORKDIR /opt
RUN curl -fsSL https://archive.apache.org/dist/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz \
  | tar xz && ln -s /opt/apache-tomcat-${TOMCAT_VER} /opt/tomcat

# Deploy WAR as ROOT
WORKDIR /opt/tomcat/webapps
COPY /src/target/java-tomcat-maven-example.war ./ROOT.war

# Foreground mode (don’t use startup.sh which forks)
EXPOSE 8080
CMD ["/opt/tomcat/bin/catalina.sh", "run"]


Fixes: uses catalina.sh run (keeps PID 1 alive), avoids chmod 777, pins Tomcat version, cleans apt cache.

Don’t ship docs

Use a .dockerignore to exclude Markdown and explanation folders from the build context:

*.md
Dockerfile Explanation/
.git
target
.vscode
.idea
.DS_Store

Build & run
docker build -t my-tomcat-app .
docker run --rm -p 8080:8080 my-tomcat-app

TL;DR

Prefer multi-stage with official Maven & Tomcat images → smaller, safer, faster.

Use catalina.sh run (not startup.sh) so the container stays up.

Exclude docs via .dockerignore.

Pin versions and avoid chmod 777.
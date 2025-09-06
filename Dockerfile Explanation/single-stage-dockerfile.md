# 🐳 Single-stage Dockerfile (Teaching Demo)

This demo shows how to build and run everything in **one image**. It’s simple, but produces a much larger image than multi-stage builds.

---

```dockerfile
# Single-stage: everything in one image (teaching demo)
FROM ubuntu:22.04
```

👉 Use Ubuntu 22.04 as the **base image**.  
This means we start with a full OS environment (heavier than slim/Alpine).

---

```dockerfile
# 1) Tools to BUILD and RUN
RUN apt-get update -y  && apt-get install -y default-jdk maven wget ca-certificates curl  && rm -rf /var/lib/apt/lists/*
```

🔧 **What this does:**  
- `apt-get update -y` → refresh package metadata.  
- `default-jdk` → install Java compiler/runtime (needed to build + run).  
- `maven` → build tool for Java projects.  
- `wget` + `curl` + `ca-certificates` → secure download tools.  
- `rm -rf /var/lib/apt/lists/*` → clean up apt cache (saves ~20–40MB).  

---

```dockerfile
# 2) Build the WAR (host -> image)
WORKDIR /src
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package
# WAR produced at /src/target/java-tomcat-maven-example.war
```

📦 **Build steps:**  
- `WORKDIR /src` → set build workspace.  
- `COPY pom.xml .` → copy dependency config first (better cache).  
- `mvn dependency:go-offline` → download dependencies ahead of time.  
- `COPY src ./src` → bring in source code.  
- `mvn package -DskipTests` → build WAR file.  

Output WAR: `/src/target/java-tomcat-maven-example.war`

---

```dockerfile
# 3) Install Tomcat (pin version)
ENV TOMCAT_VER=9.0.8
WORKDIR /opt
RUN wget -q https://archive.apache.org/dist/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz  && tar -xzf apache-tomcat-${TOMCAT_VER}.tar.gz -C /opt  && ln -sfn /opt/apache-tomcat-${TOMCAT_VER} /opt/tomcat
```

🛠️ **Tomcat setup:**  
- `ENV TOMCAT_VER=9.0.8` → version pinning (reproducible builds).  
- `WORKDIR /opt` → install Tomcat here.  
- `wget` + `tar` → download + extract Tomcat.  
- `ln -sfn` → symlink `/opt/tomcat` → `/opt/apache-tomcat-9.0.8`.  
  This allows upgrades without changing paths.  

⚠️ Could also add `rm apache-tomcat-${TOMCAT_VER}.tar.gz` to save space.

---

```dockerfile
# 4) Non-root user and ownership (safer than chmod 777)
RUN groupadd -r tomcat  && useradd -r -g tomcat -d /opt/tomcat -s /usr/sbin/nologin tomcat  && chown -R tomcat:tomcat /opt/apache-tomcat-${TOMCAT_VER}
```

🔒 **Security best practice:**  
- Create `tomcat` group & user (no login shell).  
- Change ownership of Tomcat install → prevents running as root.  

---

```dockerfile
# 5) Deploy WAR as ROOT (served at "/")
WORKDIR /opt/tomcat/webapps
RUN rm -rf ROOT
RUN cp /src/target/java-tomcat-maven-example.war ./ROOT.war
```

🚀 **Deploy the app:**  
- Switch to Tomcat’s deployment folder (`/opt/tomcat/webapps`).  
- Remove default ROOT app.  
- Copy built WAR into ROOT.war → Tomcat serves it at **http://localhost:8080/**  

---

```dockerfile
# 6) Run Tomcat in foreground (container-friendly)
EXPOSE 8080
USER tomcat
CMD ["/opt/tomcat/bin/catalina.sh", "run"]
```

🌐 **Runtime setup:**  
- `EXPOSE 8080` → documents that app listens on port 8080.  
- `USER tomcat` → drop root privileges.  
- `CMD catalina.sh run` → run Tomcat in foreground (container best practice).  

---

## ⚖️ Key Takeaways

- ✅ Easy to write: everything in one image.  
- ❌ Heavy (~700MB+): ships JDK, Maven, Tomcat, sources, build cache.  
- ❌ Larger attack surface: contains tools not needed in production.  

👉 Compare this with a **multi-stage Dockerfile**: final image is smaller, safer, and faster to ship.


🧪 Single-stage (Ubuntu only) — build + Tomcat runtime in one image
```sh
# Single-stage: everything in one image (teaching demo)
FROM ubuntu:22.04

# 1) Tools to BUILD and RUN
RUN apt-get update -y \
 && apt-get install -y default-jdk maven wget ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*

# 2) Build the WAR (host -> image)
WORKDIR /src
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package
# WAR produced at /src/target/java-tomcat-maven-example.war

# 3) Install Tomcat (pin version)
ENV TOMCAT_VER=9.0.8
WORKDIR /opt
RUN wget -q https://archive.apache.org/dist/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz \
 && tar -xzf apache-tomcat-${TOMCAT_VER}.tar.gz -C /opt \
 && ln -sfn /opt/apache-tomcat-${TOMCAT_VER} /opt/tomcat

# 4) Non-root user and ownership (safer than chmod 777)
RUN groupadd -r tomcat \
 && useradd -r -g tomcat -d /opt/tomcat -s /usr/sbin/nologin tomcat \
 && chown -R tomcat:tomcat /opt/apache-tomcat-${TOMCAT_VER}

# 5) Deploy WAR as ROOT (served at "/")
WORKDIR /opt/tomcat/webapps
RUN rm -rf ROOT
RUN cp /src/target/java-tomcat-maven-example.war ./ROOT.war

# 6) Run Tomcat in foreground (container-friendly)
EXPOSE 8080
USER tomcat
CMD ["/opt/tomcat/bin/catalina.sh", "run"]

```

🍎 vs 🍎 Multi-stage (Ubuntu only in both stages)

(This mirrors the single-stage, but splits build/runtime for size & security benefits—still no maven/tomcat base images, just Ubuntu in both stages.)
```sh
# --- Stage 1: build on Ubuntu ---
FROM ubuntu:22.04 AS build
RUN apt-get update -y \
 && apt-get install -y default-jdk maven ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package
# WAR: /usr/src/app/target/java-tomcat-maven-example.war

# --- Stage 2: runtime on Ubuntu (no Maven here) ---
FROM ubuntu:22.04
RUN apt-get update -y \
 && apt-get install -y default-jdk wget ca-certificates \
 && rm -rf /var/lib/apt/lists/*

ENV TOMCAT_VER=9.0.8
WORKDIR /opt
RUN wget -q https://archive.apache.org/dist/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz \
 && tar -xzf apache-tomcat-${TOMCAT_VER}.tar.gz -C /opt \
 && ln -sfn /opt/apache-tomcat-${TOMCAT_VER} /opt/tomcat

RUN groupadd -r tomcat \
 && useradd -r -g tomcat -d /opt/tomcat -s /usr/sbin/nologin tomcat \
 && chown -R tomcat:tomcat /opt/apache-tomcat-${TOMCAT_VER}

WORKDIR /opt/tomcat/webapps
RUN rm -rf ROOT
COPY --from=build /usr/src/app/target/java-tomcat-maven-example.war ./ROOT.war

EXPOSE 8080
USER tomcat
CMD ["/opt/tomcat/bin/catalina.sh", "run"]
```

🧾 .dockerignore (use for both; keeps builds fast/clean)
```
# build outputs and VCS/IDE noise
target
.git
.gitignore
.vscode
.idea
.DS_Store

# docs (optional)
*.md
docs/
Dockerfile Explanation/
README.md
```
🎓 What to demonstrate in class

Single-stage (Ubuntu)

Pros: simple mental model; one image does it all.

Cons: bigger image; ships Maven + build cache to prod; larger attack surface.

Multi-stage (Ubuntu→Ubuntu)

Pros: smaller final image; no Maven in runtime; better security & faster pulls.

Cons: slightly more Dockerfile complexity.

▶️ Build & Run (both)
```sh
# build
docker build -t java-tomcat-single -f Dockerfile.single .
docker build -t java-tomcat-multi  -f Dockerfile.multi  .

# run
docker run --rm -p 8080:8080 java-tomcat-single
# or
docker run --rm -p 8080:8080 java-tomcat-multi
```

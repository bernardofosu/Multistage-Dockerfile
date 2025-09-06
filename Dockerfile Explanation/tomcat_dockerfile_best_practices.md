# ☕ Tomcat Dockerfile Deep Dive with Best Practices

This document explains **each line of your Ubuntu-based Tomcat + Maven Dockerfile**, why `catalina.sh run` is better than `startup.sh`, the importance of **pinning versions**, and why to **avoid chmod 777**. It also covers symlinks, deployment paths, and WAR naming conventions. 🚀

---

## 🔍 Line-by-line explanation

```dockerfile
FROM ubuntu:22.04
```
Use Ubuntu 22.04 as the base OS.

```dockerfile
RUN apt-get update -y &&     apt-get install -y curl ca-certificates default-jdk maven &&     rm -rf /var/lib/apt/lists/*
```
- Update apt metadata.  
- Install build/runtime tools:  
  - `curl` & `ca-certificates` → secure downloads.  
  - `default-jdk` → compile/run Java.  
  - `maven` → build your WAR.  
- Clean apt cache to shrink image size.

```dockerfile
WORKDIR /src
```
Set working directory for build steps.

```dockerfile
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
```
- Copy `pom.xml` first for dependency cache reuse.  
- Pre-download deps offline (faster builds).

```dockerfile
COPY src ./src
RUN mvn -q -DskipTests package
```
- Copy source code.  
- Compile/package into a WAR (in `target/`).  
- Skip tests for speed (CI/CD should run them separately).

```dockerfile
ENV TOMCAT_VER=9.0.89
WORKDIR /opt
```
- Pin Tomcat version (`9.0.89`).  
- Use `/opt` as Tomcat install dir.

```dockerfile
RUN curl -fsSL https://archive.apache.org/dist/tomcat/tomcat-9/v${TOMCAT_VER}/bin/apache-tomcat-${TOMCAT_VER}.tar.gz   | tar xz && ln -s /opt/apache-tomcat-${TOMCAT_VER} /opt/tomcat
```
- Download/extract **that exact Tomcat version**.  
- Create symlink `/opt/tomcat` → `/opt/apache-tomcat-9.0.89`.  
- ✅ Benefit: stable path + easy upgrades.

```dockerfile
WORKDIR /opt/tomcat/webapps
COPY /src/target/java-tomcat-maven-example.war ./ROOT.war
```
- Switch to Tomcat’s deployment folder.  
- Copy WAR and rename to `ROOT.war` → app deployed at `/`.

```dockerfile
EXPOSE 8080
CMD ["/opt/tomcat/bin/catalina.sh", "run"]
```
- Document port 8080.  
- Start Tomcat in **foreground mode** with `catalina.sh run`.

---

## 🆚 `catalina.sh run` vs `startup.sh`

- **`startup.sh`**  
  - Calls `catalina.sh start`.  
  - **Forks** Tomcat to background → script exits.  
  - Bad for Docker: PID 1 dies, container stops.  

- **`catalina.sh run`**  
  - Runs Tomcat **in foreground**.  
  - PID 1 stays alive → container runs until Tomcat stops.  
  - Handles signals (e.g., `docker stop`).  
  - Logs to stdout/stderr → `docker logs` works.

👉 In containers, always use **foreground** mode.

---

## 📌 Pinning versions (why important)

- **Reproducibility** → same build every time.  
- **Security** → control exactly what version you run.  
- **Predictability** → no silent upgrades breaking your app.  

Examples:  
- `ENV TOMCAT_VER=9.0.89` instead of "latest".  
- `FROM ubuntu:22.04` instead of `FROM ubuntu:latest`.  

Check version with:  
```sh
/opt/tomcat/bin/version.sh
```

---

## 🚫 Why avoid `chmod 777`

- Grants **rwx to everyone** → insecure.  
- Masks ownership/permissions problems.  
- Containers usually only need minimal access.

✅ Better:  
- `chown -R tomcat:tomcat /opt/tomcat`  
- Config dirs → `chmod 750`  
- Logs/temp/work → `chmod 770`  

Even better: run Tomcat as non-root:  
```dockerfile
USER tomcat
```

---

## 🔗 Symlink vs Rename

### Rename (simpler)
```sh
mv apache-tomcat-9.0.89 tomcat
```
- One folder: `/opt/tomcat`.  
- No version trace.

### Symlink (recommended)
```sh
ln -s /opt/apache-tomcat-9.0.89 /opt/tomcat
```
- Keeps `/opt/apache-tomcat-9.0.89`.  
- `/opt/tomcat` points there.  
- Upgrading = just repoint symlink.  
- Easier rollback.

---

## 📂 Deployment path & WAR naming

- Tomcat auto-deploys anything in `/opt/tomcat/webapps/`.  
- `hello.war` → served at `/hello/`.  
- `ROOT.war` → served at `/`.  
- Exploded folder `myapp/` → served at `/myapp/`.

✅ In Docker: renaming to `ROOT.war` is common → single app per container, clean root URL.

---

## 🧭 Additional Best Practices

- **Multi-stage builds** → build with `maven:` then copy WAR into `tomcat:` image. Smaller & safer.  
- **.dockerignore** → exclude `*.md`, `.git`, `target/`, `docs/`, IDE files.  
- **Use official Tomcat images** → already tuned for foreground + correct layout.  

---

## ✅ TL;DR

- Use `catalina.sh run` → foreground process, logs, signals.  
- Pin versions (`TOMCAT_VER=9.0.89`, `ubuntu:22.04`).  
- Avoid `chmod 777` → fix ownership/least-privilege.  
- Symlink gives upgrade flexibility.  
- `ROOT.war` = app at `/` (best for Docker).  
- Multi-stage builds → tiny, production-ready images.  

# 🐳 Dockerfile Multi-Stage Build – Notes

## 📦 Goal
Build a Java web application using Maven, then deploy it on Tomcat.  
Exclude unnecessary files (like Markdown `.md` docs or explanation folders) from the Docker image.  

---

## 1️⃣ Example Multi-Stage Dockerfile
```dockerfile
# ---------- Stage 1: Build ----------
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /usr/src/app

# copy only pom.xml first to cache dependencies
COPY pom.xml .
RUN mvn -q -e -DskipTests dependency:go-offline

# copy application source code
COPY src ./src

# build WAR
RUN mvn -q -DskipTests package

# ---------- Stage 2: Runtime (Tomcat) ----------
FROM tomcat:9.0-jdk17-temurin
WORKDIR /usr/local/tomcat/webapps

# copy only the built artifact from Stage 1
COPY --from=build /usr/src/app/target/java-tomcat-maven-example.war ./ROOT.war
```

- Stage 1 (`maven`) → compiles the project and produces a `.war`.  
- Stage 2 (`tomcat`) → only contains Tomcat + final `.war`.  

✅ Result → smaller, cleaner image.  

---

## 2️⃣ Exclude Files with `.dockerignore`
To keep `.md` and docs out of the build context, create a `.dockerignore` file in the project root:

```
# exclude documentation & markdown
*.md
Dockerfile Explanation/

# common excludes
.git
.gitignore
node_modules
target
.idea
.vscode
.DS_Store
```

This ensures Markdown files and the `Dockerfile Explanation/` folder **aren’t even sent to Docker** during build.  

---

## 3️⃣ Build & Run
```bash
# build image
docker build -t my-tomcat-app .

# run container
docker run --rm -p 8080:8080 my-tomcat-app
```

Tomcat will serve the app on **http://localhost:8080**.  

---

## ⚡ Benefits
- 🚀 Faster builds (no junk files copied).  
- 📉 Smaller image size (only WAR + Tomcat).  
- 🔒 Cleaner, more secure production container.  

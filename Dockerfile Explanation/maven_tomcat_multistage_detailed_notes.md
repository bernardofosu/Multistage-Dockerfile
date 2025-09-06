# 🐳 Maven → Tomcat Multi‑Stage Docker Build — Deep, Line‑by‑Line Notes

These notes fully explain the multi‑stage Dockerfile you’re using, how it maps to a manual Tomcat install on Linux, why we rename to **`ROOT.war`**, and how to exclude files from the build context with **`.dockerignore`**. It also includes build/run steps, verification, and troubleshooting.


---

## 📄 The Dockerfile (full, as discussed)

```dockerfile
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
```
> ✅ This creates a small, production‑friendly image that contains **only Tomcat + your WAR**. No Maven, no source code, no docs.


---

## 🧠 Line‑by‑Line Explanation (What/Why/How)

### 🏗 Stage 1 — Build (Maven + JDK)
```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
```
- **What:** Start from the official **Maven** image bundled with **Eclipse Temurin JDK 17**.  
- **Why:** It already has Maven + JDK installed → minimal setup to compile your app.  
- **How:** `AS build` names this stage; later we copy artifacts from it with `--from=build`.

```dockerfile
WORKDIR /usr/src/app
```
- **What:** Set the working directory for subsequent commands in this stage.  
- **Why:** Keeps paths predictable and builds readable.  
- **How:** All following commands run under `/usr/src/app`.

```dockerfile
# Cache deps
COPY pom.xml .
```
- **What:** Copy **only** the `pom.xml` first.  
- **Why:** Enables **layer caching** for dependencies. Maven deps rarely change unless the POM does, so this speeds up rebuilds dramatically.  
- **How:** If your source code changes but `pom.xml` doesn’t, Docker won’t re‑download all dependencies.

```dockerfile
RUN mvn -q -DskipTests dependency:go-offline
```
- **What:** Pre‑fetch all dependencies required by the POM into Maven’s local cache.  
- **Why:** Ensures faster builds and lets the next `mvn package` run without re‑resolving deps; improves determinism in CI.  
- **Flags:**  
  - `-q` → quiet logs.  
  - `-DskipTests` → do not run tests during dependency resolution (faster).

```dockerfile
# Build app
COPY src ./src
```
- **What:** Copy your Java sources into the image.  
- **Why:** We do this **after** dependency caching so changing code doesn’t invalidate the deps layer.

```dockerfile
RUN mvn -q -DskipTests package
```
- **What:** Compile and package the app into a `.war` (e.g., `target/java-tomcat-maven-example.war`).  
- **Why:** Produces the deployable artifact. Tests are skipped for speed (enable them in CI as needed).


### 🚀 Stage 2 — Runtime (Tomcat only)
```dockerfile
FROM tomcat:9.0-jdk17-temurin
```
- **What:** Start from the official **Tomcat 9** image with JDK 17.  
- **Why:** Clean runtime; no build tools. Fewer CVEs, smaller image, faster pulls/run.

```dockerfile
WORKDIR /usr/local/tomcat/webapps
```
- **What:** Switch to Tomcat’s **deployment directory** in the official image.  
- **Why:** Tomcat **auto‑deploys** any `.war` dropped into `webapps/` at startup.

```dockerfile
# Deploy as ROOT.war
COPY --from=build /usr/src/app/target/java-tomcat-maven-example.war ./ROOT.war
```
- **What:** Copy the built WAR out of Stage 1 into this stage’s `webapps/` folder and **rename to `ROOT.war`**.  
- **Why:** Tomcat deploys `ROOT.war` at the **root context** → `http://host:8080/`.  
  - If you keep the original name, your app lives at `http://host:8080/java-tomcat-maven-example/` (context path = WAR name).  
- **How:** `--from=build` references the named builder stage; the `WORKDIR` ensures `./ROOT.war` lands in `webapps/`.

```dockerfile
# (Optional) health: Tomcat listens on 8080
EXPOSE 8080
```
- **What:** Document that the container listens on port **8080**.  
- **Why:** Helps tooling (Docker/K8s) know which port to map. (EXPOSE doesn’t open the port by itself.)


---

## 🧾 `.dockerignore` — Keep Docs & Noise Out

Create a `.dockerignore` file at the project root to avoid bloating the image and context:

```gitignore
# docs & markdown
*.md
docs/
Dockerfile Explanation/

# build & VCS noise
target
.git
.gitignore
.vscode
.idea
.DS_Store
node_modules
```
- **Why:** Anything in the build **context** can be copied into the image by accident and slows down `docker build`.  
- **Result:** Faster builds, smaller images, fewer surprises.


---

## 🐧 Mapping to a Manual Linux Install (Bare Metal)

On a server where Tomcat lives at `/opt/apache-tomcat-9.0.65`:

```bash
# 1) Build the WAR
mvn -q -DskipTests package

# 2) Deploy to Tomcat's webapps/ as ROOT (root context)
cp target/java-tomcat-maven-example.war    /opt/apache-tomcat-9.0.65/webapps/ROOT.war
```
- **Same logic** as the Dockerfile: put the WAR in Tomcat’s **`webapps/`** folder.  
- Rename to **`ROOT.war`** to serve at `/`.  
- If you don’t rename it, you’ll get `/java-tomcat-maven-example/`.


### 📍 Paths Quick Reference
| Environment | Tomcat Home | Deploy Folder |
|---|---|---|
| **Official Docker image** | `/usr/local/tomcat` | `/usr/local/tomcat/webapps/` |
| **Manual install (example)** | `/opt/apache-tomcat-9.0.65` | `/opt/apache-tomcat-9.0.65/webapps/` |


---

## ▶️ Build & Run the Container

```bash
# Build the image
docker build -t my-tomcat-app .

# Run: map host 8080 -> container 8080
docker run --rm -p 8080:8080 my-tomcat-app

# Verify HTTP
curl -I http://localhost:8080/
```

### 🔎 Check inside the container
```bash
# find your container id
docker ps

# stream logs (look for "Deployment of web application archive")
docker logs -f <container_id>

# open a shell to inspect
docker exec -it <container_id> bash
ls -lah /usr/local/tomcat/webapps
```


---

## 🔄 Alternative (Quick Testing): Mount WAR at Runtime

If you already have the WAR locally and just want to test it without rebuilding the image each time:

```bash
# build locally on your machine
mvn -q -DskipTests package

# run official Tomcat and mount the WAR as ROOT
docker run --rm -p 8080:8080   -v "$(pwd)/target/java-tomcat-maven-example.war:/usr/local/tomcat/webapps/ROOT.war"   tomcat:9.0-jdk17-temurin
```
- **Pros:** Fast iteration.  
- **Cons:** Depends on external volume; not ideal for immutable, reproducible production images.


---

## ⚠️ Common Pitfalls & How to Avoid Them

- **WAR copied to the wrong folder** → `/usr/local/tomcat/` instead of `/usr/local/tomcat/webapps/`.  
  - ✅ Fix: always deploy to `webapps/`.  
- **Forgetting `ROOT.war`** when you expect root URL.  
  - ✅ Fix: rename to `ROOT.war` for `/`.  
- **Massive images** because you shipped Maven/JDK/source in final image.  
  - ✅ Fix: use multi‑stage; final stage is Tomcat runtime only.  
- **Slow rebuilds** due to bad caching.  
  - ✅ Fix: copy `pom.xml` first; run `dependency:go-offline`; copy `src` later.  
- **Using `startup.sh`** in containers (it forks and exits).  
  - ✅ Use the official image; it runs Tomcat in the foreground (`catalina.sh run`).  
- **Port not reachable** from host.  
  - ✅ Map ports: `docker run -p 8080:8080 …`; ensure security groups/firewalls allow 8080.  
- **Docs and extra folders baked into image**.  
  - ✅ Add a `.dockerignore`.


---

## ✅ The “Correct” COPY (Root Context in Docker)

```dockerfile
COPY --from=build /usr/src/app/target/java-tomcat-maven-example.war /usr/local/tomcat/webapps/ROOT.war
```
- **Guarantee:** Tomcat **sees** it, **unpacks** it, and serves it at:  
  👉 **http://localhost:8080/**


---

## 🧪 Troubleshooting Checklist

- Is the WAR present in `webapps/`?  
- Did logs show something like *“Deployment of web application archive … has finished”*?  
- Did you map the port (`-p 8080:8080`)?  
- Are you expecting `/` but see 404? → did you rename to `ROOT.war`?  
- Is the WAR actually produced (`ls target/` after `mvn package`)?  
- Any stack traces in `docker logs -f <id>`?


---

## 📝 Bonus: Why Multi‑Stage Beats Single‑Stage

- **Smaller final image** → only Tomcat + WAR.  
- **Fewer CVEs** → no compilers/build tools in prod image.  
- **Faster CI/CD** → better caching with `pom.xml` then `src`.  
- **Cleaner separation of concerns** → build vs. runtime.


---

## 📚 Appendix — Minimal `.dockerignore` & Commands

**`.dockerignore`**
```gitignore
*.md
docs/
Dockerfile Explanation/
target
.git
.gitignore
.vscode
.idea
.DS_Store
node_modules
```

**Build & Run**
```bash
docker build -t my-tomcat-app .
docker run --rm -p 8080:8080 my-tomcat-app
```

**Manual Linux Deploy (no Docker)**
```bash
mvn -q -DskipTests package
cp target/java-tomcat-maven-example.war /opt/apache-tomcat-9.0.65/webapps/ROOT.war
```

---

If you want, I can also add a **GitHub Actions** workflow that builds this image and pushes it to **Docker Hub/ECR**, plus a **Kubernetes Deployment** manifest to run it. 🚀

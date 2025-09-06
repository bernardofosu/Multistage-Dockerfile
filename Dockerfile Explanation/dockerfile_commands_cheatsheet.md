# 🐳 Dockerfile Commands – Cheat Sheet

### 1️⃣ **FROM** – Base Image
```dockerfile
FROM ubuntu:22.04
```
- 📌 Defines the **base image**.  
- Every Dockerfile starts with `FROM` (unless `scratch` is used).  

---

### 2️⃣ **RUN** – Execute Build Commands
```dockerfile
RUN apt-get update && apt-get install -y curl
```
- ⚡ Executes commands **at build time**.  
- Each `RUN` creates a new **layer** in the image.  

---

### 3️⃣ **WORKDIR** – Set Working Directory
```dockerfile
WORKDIR /app
```
- 📂 Sets the directory for subsequent commands.  
- Auto-creates the folder if it doesn’t exist.  

---

### 4️⃣ **COPY** – Copy Files into Image
```dockerfile
COPY src/ /app/src/
```
- 📥 Copies files/folders from **host → image**.  
- Best for static files, configs, app code.  

---

### 5️⃣ **ADD** – Copy with Extras
```dockerfile
ADD app.tar.gz /app/
```
- 📦 Like `COPY`, but can:  
  - Auto-extract archives (`.tar.gz`).  
  - Fetch remote URLs.  
- ⚠️ Use `COPY` unless you need these extras.  

---

### 6️⃣ **ENV** – Environment Variables
```dockerfile
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```
- 🌍 Sets variables for the container environment.  
- Available to all `RUN`, `CMD`, `ENTRYPOINT`.  

---

### 7️⃣ **EXPOSE** – Document Ports
```dockerfile
EXPOSE 8080
```
- 🌐 Documents that the container listens on **8080**.  
- 🚫 Doesn’t publish the port (use `docker run -p 8080:8080`).  

---

### 8️⃣ **CMD** – Default Command
```dockerfile
CMD ["java", "-jar", "app.jar"]
```
- ▶️ Defines the **default runtime command**.  
- Only the **last CMD** is used.  
- Can be overridden by `docker run <image> <command>`.  

---

### 9️⃣ **ENTRYPOINT** – Main Process
```dockerfile
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```
- 🔑 Defines the **main executable**.  
- Unlike `CMD`, it’s **not overridden** by default.  
- Often used with `CMD` for flexible arguments.  

---

### 🔟 **USER** – Set User
```dockerfile
USER tomcat
```
- 👤 Switches from root to another user.  
- 🔒 Best practice: run apps as non-root.  

---

### 1️⃣1️⃣ **ARG** – Build-Time Variables
```dockerfile
ARG VERSION=1.0
RUN echo "Building version $VERSION"
```
- 🛠️ Variables available **during build only**.  
- Passed with: `docker build --build-arg VERSION=2.0 .`  

---

### 1️⃣2️⃣ **VOLUME** – Persistent Storage
```dockerfile
VOLUME ["/data"]
```
- 💾 Creates a mount point for external volumes.  
- Keeps data outside the container lifecycle.  

---

### 1️⃣3️⃣ **LABEL** – Metadata
```dockerfile
LABEL maintainer="you@example.com" version="1.0"
```
- 🏷️ Adds metadata (maintainer, version, description).  

---

### 1️⃣4️⃣ **HEALTHCHECK** – Health Monitoring
```dockerfile
HEALTHCHECK CMD curl --fail http://localhost:8080/ || exit 1
```
- ❤️ Checks if the container is healthy.  
- Docker marks container as `healthy` or `unhealthy`.  

---

### 1️⃣5️⃣ **ONBUILD** – Trigger in Child Images
```dockerfile
ONBUILD RUN echo "Runs when image is used as a base"
```
- 🔄 Executes only when this image is used as a **base** for another build.  
- Useful for base/development images.  

---

# ✅ TL;DR Summary
- **FROM** → base image  
- **RUN** → build commands  
- **WORKDIR** → working dir  
- **COPY / ADD** → bring files in  
- **ENV** → set environment vars  
- **EXPOSE** → document port  
- **CMD** → default container command  
- **ENTRYPOINT** → main process  
- **USER** → non-root execution  
- **ARG** → build-time vars  
- **VOLUME** → persistent storage  
- **LABEL** → metadata  
- **HEALTHCHECK** → monitor app health  
- **ONBUILD** → deferred triggers  

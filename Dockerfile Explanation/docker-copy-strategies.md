# 🐳 Docker COPY Strategies — Selective vs. Context Copy

## 🔎 What you’re comparing
**Selective COPY (what you have):**
```dockerfile
# Build stage
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline

COPY src ./src
RUN mvn -q -DskipTests package
```
- ✅ Copies only what the build needs, in a **stable order** (POM first, then `src`) → **great caching**.
- ✅ Final stage copies **only the WAR** → **tiny, safer runtime** image.

**`COPY . .` with `.dockerignore`:**
```dockerfile
COPY . .
```
`.dockerignore`:
```
*.md
Dockerfile Explanation/
.git
target
.vscode
.idea
.DS_Store
```
- ✅ Simpler, but **pulls everything** else in the context (except ignored).
- ⚠️ One small unrelated change (e.g., a config/script) can **bust the cache** for your dependency layer → **slower builds**.
- ⚠️ Higher chance of accidentally **including files** you forgot to ignore (secrets, dev scripts, etc.).

---

## ⚖️ Practical differences

| Aspect | Selective `COPY` (pom.xml → deps, then src) | `COPY . .` + `.dockerignore` |
|---|---|---|
| **Build speed (cache)** | ⭐ **Best**: deps cached until `pom.xml` changes | ❌ **Worse**: many files can invalidate cache |
| **Image size** | **Small final** (multi-stage copies only WAR) | Similar final size if multi-stage, but easier to leak junk into build stage |
| **Safety** | **High**: only intended files included | **Lower**: easy to miss ignore entries / leak creds |
| **Reproducibility** | **High**: explicit inputs | **Lower**: depends on state of whole context |
| **Simplicity** | Slightly more verbose | Simple, but **foot‑guns** |

---

## ✅ Recommended pattern (best of both)
- Keep your **selective `COPY`** steps for Maven **caching**.
- Also keep a **`.dockerignore`** to trim the build context (speeds up sends to daemon & prevents accidental copies).

**Minimal `.dockerignore`:**
```
# docs & local noise
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
```

---

## 🧠 Bottom line
**Winner:** **Selective `COPY` (+ multi-stage)** **with** a `.dockerignore`.

Use `COPY . .` only for **tiny prototypes** where caching, safety, and determinism don’t matter as much.

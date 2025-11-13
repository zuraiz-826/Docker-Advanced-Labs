Here you go — a **beautiful, fully formatted, emoji-enhanced Markdown version** of your entire lab.
Clean, structured, and visually engaging. 🚀🐳✨

---

# **🐳 Lab 2: Understanding Docker Images**

## **🎯 Lab Objectives**

By the end of this lab, you will be able to:

* 🧩 Understand Docker images and their role in containerization
* 🔍 Search Docker Hub for images
* ⬇️ Pull Docker images locally
* 📁 List, inspect, and manage Docker images
* 🧹 Remove unused images to save space
* 🏷️ Use image tags effectively
* 🧱 Explore the layered structure of images
* 🛠️ Apply best practices for image management

---

## **📚 Prerequisites**

Before beginning, ensure you have:

* 💻 Basic CLI knowledge
* 🐳 Completed **Lab 1** or installed Docker
* 🐧 Familiarity with Linux commands
* 📦 Conceptual understanding of containers

---

## **🖥️ Lab Environment Setup**

🎉 **Good News!**
Al Nafi provides ready-made cloud Linux machines with Docker pre-installed.

Your lab environment includes:

* Ubuntu OS 🐧
* Docker Engine ⚙️
* Terminal with sudo access
* Internet connectivity 🌐

---

# **🧭 Task 1: Exploring Docker Hub**

## **📌 Subtask 1.1: What is Docker Hub?**

Docker Hub is the **largest repository** of container images — think of it as an **app store for Docker**.

### **Step 1:**

Open 👉 [https://hub.docker.com](https://hub.docker.com)

### **Step 2:**

Explore the interface:

* 🔎 Search bar
* ⭐ Featured repositories
* 📂 Image categories

---

## **🔍 Subtask 1.2: Searching via Web**

### **Step 1:** Search popular images

Search for **ubuntu**:

* 📄 Description
* 🏷️ Tags
* 📥 Pull command
* 📊 Download stats

### **Step 2:** Try other popular images:

* `nginx` 🌐
* `mysql` 🗄️
* `python` 🐍

---

## **💻 Subtask 1.3: Searching via CLI**

Run:

```bash
docker search ubuntu
```

**Columns Explained:**

* **NAME**: Repo name
* **DESCRIPTION**: Image summary
* **STARS ⭐**: Community rating
* **OFFICIAL**: Verified image
* **AUTOMATED**: Auto-built

Try more:

```bash
docker search nginx
docker search --limit 5 python
```

---

# **⬇️ Task 2: Pulling Docker Images**

## **🏷️ Subtask 2.1: Understanding Tags**

Tags = versions.
`latest` is convenient but ❌ NOT ideal for production.

---

## **📥 Subtask 2.2: Pulling Ubuntu**

### **Step 1:** Pull latest

```bash
docker pull ubuntu
```

### **Step 2:** Watch layers ⬇️

Each layer = filesystem change.

### **Step 3:** Pull specific versions

```bash
docker pull ubuntu:20.04
docker pull ubuntu:22.04
```

---

## **📦 Subtask 2.3: Pulling Other Images**

```bash
docker pull nginx:alpine
docker pull python:3.9-slim
```

**Why these tags?**

* 🪶 `alpine` → Tiny & secure
* 🔧 `slim` → Lightweight
* 🔢 `3.9` → Specific runtime version

---

# **📁 Task 3: Managing Docker Images**

## **📝 Subtask 3.1: Listing Images**

```bash
docker images
```

**Columns:**

* REPOSITORY 🏷️
* TAG
* IMAGE ID 🔑
* CREATED
* SIZE

Formatted list:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

---

## **🎯 Subtask 3.2: Filtering**

```bash
docker images ubuntu
docker images -q
docker images -a
```

---

## **🧹 Subtask 3.3: Removing Images**

Remove by name:

```bash
docker rmi ubuntu:20.04
```

Remove by ID:

```bash
docker images python:3.9-slim
docker rmi <IMAGE_ID>
```

Force remove:

```bash
docker rmi -f nginx:alpine
```

Remove unused:

```bash
docker image prune
```

Remove ALL (⚠️ dangerous):

```bash
docker rmi $(docker images -q)
```

---

# **🏷️ Task 4: Working with Tags**

## **📌 Subtask 4.1: Why Tags Matter**

* Ensures **reproducibility**
* Avoids breaking changes
* Allows optimized variants

---

## **▶️ Subtask 4.2: Running Containers with Tags**

Pull:

```bash
docker pull ubuntu:18.04
docker pull ubuntu:20.04
docker pull ubuntu:22.04
```

Run:

```bash
docker run -it ubuntu:18.04 /bin/bash
cat /etc/os-release
exit
```

Compare with 22.04 👇

---

## **🛠️ Subtask 4.3: Creating Custom Tags**

Tagging:

```bash
docker tag ubuntu:22.04 my-ubuntu:production
docker images | grep my-ubuntu
```

---

# **🧱 Task 5: Inspecting Image Layers**

## **📘 Subtask 5.1: Layers Explained**

Docker images = stacks of layers.

Benefits:

* 🔁 Shared layers
* ⚡ Faster builds
* 💾 Reduced storage

---

## **🔍 Subtask 5.2: Inspecting Images**

```bash
docker inspect ubuntu:22.04
```

Extract fields:

```bash
docker inspect --format='{{.Architecture}}' ubuntu:22.04
docker inspect --format='{{.Created}}' ubuntu:22.04
docker inspect --format='{{.Size}}' ubuntu:22.04
```

---

## **📜 Subtask 5.3: Image History**

```bash
docker history ubuntu:22.04
docker history --no-trunc ubuntu:22.04
```

Compare:

```bash
docker history nginx:alpine
docker history python:3.9-slim
```

---

## **📊 Subtask 5.4: Layer Efficiency**

Pull Node:

```bash
docker pull node:16
```

Compare histories:

```bash
docker history node:16
docker history ubuntu:22.04
```

---

# **🧹 Task 6: Best Practices & Cleanup**

## **🌟 Subtask 6.1: Best Practices**

* Use **specific tags**
* Clean unused images regularly
* Choose minimal images (`alpine`, `slim`)
* Know your layers

---

## **🧼 Subtask 6.2: Cleanup**

Disk usage:

```bash
docker system df
```

Clean images:

```bash
docker image prune
```

Aggressive cleanup (⚠️ careful):

```bash
docker system prune -a
```

---

## **📈 Subtask 6.3: Monitoring**

Largest images:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -h
```

---

# **🛠️ Troubleshooting**

### ❌ Permission Denied

```bash
sudo docker <command>
sudo usermod -aG docker $USER
```

### ❌ Pull Failed

```bash
ping docker.io
docker search <name>
```

### ❌ Cannot Remove Image

```bash
docker ps -a
docker rm <container>
docker rmi <image>
```

### ❌ Disk Full

```bash
docker system df
docker system prune -a --volumes
```

---

# **🧪 Lab Verification**

Run:

```bash
docker images
docker inspect ubuntu:latest
docker history ubuntu:latest
docker system df
```

---

# **🎉 Conclusion**

You successfully completed **Lab 2: Understanding Docker Images**! 🚀🐳

### **🏆 What You Achieved**

* Explored Docker Hub
* Learned image management commands
* Understood layer architecture
* Practiced best practices
* Optimized storage and performance

### **🌍 Why It Matters**

* Ensures predictable deployments
* Improves build efficiency
* Reduces storage
* Enhances security

### **➡️ Next Steps**

* Build custom Dockerfiles
* Create multi-stage builds
* Secure images
* Work with private registries

You’re now ready for more advanced Docker workflows and the **DCA certification**. 💪🐳🔥

---

If you'd like, I can also:

✨ Convert this into a **PDF**
📘 Turn it into a **beautiful handout**
📝 Make a **cheat sheet**
🚀 Add diagrams

Just tell me!

# 🚀 Day 45 – Docker Build & Push using GitHub Actions

## 📌 Objective
Today I built a complete CI/CD pipeline using GitHub Actions that automatically builds a Docker image and pushes it to Docker Hub whenever code is pushed to the main branch.

---

## ⚙️ Workflow File
Path: `.github/workflows/docker-publish.yml`

```yaml
name: Day 45 - Docker Build and Push

on:
  push:
    branches:
      - main
      - feature/**

jobs:
  docker:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set short SHA
        run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> $GITHUB_ENV

      - name: Log in to Docker Hub
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: false
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:latest
            ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:sha-${{ env.SHORT_SHA }}

      - name: Build and push Docker image
        if: github.ref == 'refs/heads/main'
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:latest
            ${{ secrets.DOCKER_USERNAME }}/github-actions-practice:sha-${{ env.SHORT_SHA }}
```

---

## 🐳 Docker Hub Repository
👉 https://hub.docker.com/r/anan623/github-actions-practice

---

## 🔥 What This Pipeline Does
- Automatically triggers on code push
- Builds Docker image in GitHub Actions
- Logs in to Docker Hub securely using secrets
- Tags the image with:
  - `latest`
  - `sha-<commit>`
- Pushes image only from `main` branch

---

## 🌿 Feature Branch Testing
I tested the workflow on a feature branch:
- Docker image was successfully built ✅
- Image was NOT pushed ❌ (as expected)

---

## 🏷️ Status Badge
Added in `README.md`:

```md
![Docker Publish](https://github.com/Devel955/github-actions-practice/actions/workflows/docker-publish.yml/badge.svg)
```

---

## 🚀 Pull & Run the Image

```bash
docker pull anan623/github-actions-practice:latest
docker run -d -p 8080:80 anan623/github-actions-practice:latest
```

🌐 Access in browser:
```
http://13.53.174.111:8080
```

---

## 🔄 Full CI/CD Flow

```text
Git Push
   ↓
GitHub Actions Triggered
   ↓
Docker Image Build
   ↓
Docker Hub Push
   ↓
EC2 Pull Image
   ↓
Run Container
   ↓
Access via Browser
```

---

## 📸 Screenshots
- GitHub Actions successful workflow
- Docker Hub repository with tags
- Running application in browser

---

## 🧠 Key Learnings
- Automated Docker builds using CI/CD
- Secure authentication using GitHub Secrets
- Branch-based deployment control
- End-to-end pipeline from code to deployment

---

## ✅ Conclusion
This task helped me understand the complete real-world CI/CD pipeline — from pushing code to deploying a running container on a cloud server.

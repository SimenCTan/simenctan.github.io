---
title: 优化Gihub Action CI/CD Workflows
date: 2024-02-21 16:30:22 +0800
categories: [Infrastructure]
tags: [github action]     # TAG names should always be lowercase
mermaid: true
authors: [1,2]
---

### container构建过程
Docker 镜像最好可视化为一层堆栈，其中每一层代表由 Dockerfile 中的指令产生的一组文件系统更改。每个层仅包含其之前层的更改。这确保了跨层数据不会重复。分层结构的一大优点是中间文件系统（层）可以在后续的 Docker 构建中重用。这种层的重用构成了 Docker 缓存的基础，并导致更快的增量构建，而不是总是从头开始构建镜像。

最好的 Dockerfile 是按从最不频繁到最频繁突变的顺序堆叠层的 Dockerfile。将最不可能更改的层合并为每个 Docker 构建的基础，可以在构建后续 Docker 映像时最有效地重用层。

```yml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
```
这就是构建和推送 Docker 镜像的普通 GitHub Action 工作流程的样子。此工作流程假设 Dockerfile 位于存储库的根目录中，并使用两个流行的 Docker 操作：
`setup-buildx-action` 配置 Docker Buildx 以创建用于运行映像构建的构建器实例
`build-push-action` 使用上一步中配置的构建器构建并推送 Docker 映像
此工作流程中没有发生缓存每次运行时，Docker 都必须从头开始构建映像！

### 使用GitHub Actions 缓存
第一种缓存策略是将 Docker 层块缓存到本机 Github Actions 缓存中。这种方法是最简单的，然而随着代码库规模的扩大，这种方法也存在一些局限性。每个GitHub仓库只分配了10GB的缓存空间之后缓存中的最旧条目将被驱逐。如果您的 Docker 镜像相当大，或者有多个层，您很可能会遇到这个限制，并且无法获得有效缓存的好处。
GitHub 的缓存仅限于运行 Docker 构建的开发分支。无法通过这种方法在整个组织或与其他构建系统共享缓存层。
启用由 GitHub 缓存支持的缓存。
```yml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: user/app:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
`cache-from` 指向构建过程将尝试查找用于构建 Docker 镜像的缓存的源头
`cache-to` 指定了构建过程中生成的缓存将被存储的目的地
在上述工作流程中，第一次运行将是未缓存的运行，因为在 cache-from 中没有缓存可导入。在此运行结束时，缓存块将被写入 cache-to。随后的运行将能够利用这些缓存的 Docker 层

### 使用注册表作为缓存后端`inline`
使用 Docker 注册表作为缓存后端。Docker 注册表是用于存储和分发 Docker 镜像的系统。您可以利用注册表支持的缓存有两种方式，每种方式都有其自身的优势和局限性。
- 内联缓存
  内联缓存是设置最简单的注册表支持的缓存。这种方法将构建缓存工件直接嵌入到 Docker 镜像中。
```yml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: user/app:latest
          cache-from: type=registry,ref=user/app:latest
          cache-to: type=inline,mode=min
```
cache-to 现在指向不同的缓存后端。type=inline 指示 Docker 构建器将缓存产物嵌入到 Docker 镜像中，并将它们推送到与 Docker 镜像相同的位置。此工作流程的后续运行将使用 cache-from 中引用的镜像作为基础，从而显著加快 Docker 构建速度。相比GitHub Action缓存注册表作为缓存后端有如下优势：
1. 不受 GitHub 的缓存大小限制和清除策略的约束
2. 可以在整个组织或其他构建系统中重复使用缓存产物


多阶段 Docker 构建是将 Dockerfile 组织为多个阶段的构建。每个阶段都有自己的基础映像和指令集。有完全的灵活性来决定前一阶段的哪些工件应该复制到下一阶段。例如，您可以拥有一个包含所有工具和依赖项的构建阶段，但让您的最终阶段仅包含运行应用程序所需的部分,如下你定义的dockerfile
```yml
# Build stage
FROM node:16-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

# Final stage
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/build /app
CMD ["node", "server.js"]
```
仅将必要的文件和依赖项复制到最终的生产镜像中，使得它比将整个Dockerfile写在单个阶段中要小得多且更安全。内联缓存仅支持 mode=min。此模式仅缓存最终阶段的层，而不是多阶段Docker构建中的任何中间层。这显著降低了在后续Docker构建中缓存命中的机会。内联缓存的另一个缺点是，由于它将缓存产物嵌入到Docker镜像中，这可能会显著增加您的Docker镜像大小。接下来，我们介绍我们的最终缓存解决方案。

### 注册表缓存后端`inline cache++`
注册表缓存后端是`inline cache++`。该后端将缓存工件作为与 Docker 映像不同的单独映像推送到注册表中的专用位置。
```yml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: user/app:latest
          cache-from: type=registry,ref=user/app:buildcache
          cache-to: type=registry,ref=user/app:buildcache,mode=max
```
与内联缓存相反，当配置了 mode=max 时，注册表缓存支持在多阶段 Docker 构建中缓存中间层。这增加了后续构建中缓存命中的机会。它还支持大量选项来控制压缩类型、级别和缓存映像的命名等。虽然设置起来有些复杂，但这是 Docker 构建的最佳缓存解决方案。

# GitHub Actions

GitHub Actions 是 GitHub 内置的 CI/CD 平台，通过在 `.github/workflows/` 目录下放置 YAML 文件定义工作流，与代码仓库深度集成，支持 Push、PR、Issue、Release 等几乎所有 GitHub 事件触发。

**核心概念：**

| 概念            | 说明                                                    |
| --------------- | ------------------------------------------------------- |
| **Workflow**    | 工作流，一个 YAML 文件定义一套自动化流程                |
| **Event**       | 触发 Workflow 的事件（push、pull_request、schedule 等） |
| **Job**         | 工作流中的一个任务，默认并行执行                        |
| **Step**        | Job 中的单个步骤，顺序执行                              |
| **Action**      | 可复用的步骤单元，来自 GitHub Marketplace 或自定义      |
| **Runner**      | 执行 Job 的机器（GitHub 托管 或 Self-hosted）           |
| **Artifact**    | Job 产物，可跨 Job 或工作流共享                         |
| **Secret**      | 加密存储的敏感信息，通过 `${{ secrets.NAME }}` 引用     |
| **Variable**    | 非敏感配置变量，通过 `${{ vars.NAME }}` 引用            |
| **Environment** | 部署目标环境，可配置保护规则和审批人                    |
| **Matrix**      | 矩阵策略，用不同参数组合并行运行同一 Job                |
| **Context**     | 运行时信息（github、env、secrets、jobs 等上下文）       |

---

## Workflow 基本结构

```yaml
# .github/workflows/ci.yml
name: CI Pipeline # 工作流名称（显示在 Actions 页面）

on: # 触发事件
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env: # 全局环境变量
  APP_NAME: my-app
  REGISTRY: ghcr.io

jobs:
  build: # Job ID
    name: Build Application # Job 显示名（可选）
    runs-on: ubuntu-latest # Runner 类型

    steps:
      - name: Checkout code
        uses: actions/checkout@v4 # 使用 Action

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"

      - name: Build with Maven
        run: mvn clean package -DskipTests

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 7
```

---

## 触发事件（on）

### 代码事件

```yaml
on:
  # Push 触发
  push:
    branches:
      - main
      - "release/**"
    branches-ignore:
      - "docs/**"
    paths: # 只在这些文件变更时触发
      - "src/**"
      - "pom.xml"
    paths-ignore:
      - "**.md"
    tags:
      - "v*.*.*"

  # PR 触发
  pull_request:
    types: [opened, synchronize, reopened] # PR 事件类型
    branches: [main]

  # PR 合并到目标分支（用于 fork 仓库也能访问 secrets）
  pull_request_target:
    branches: [main]
```

### 定时触发

```yaml
on:
  schedule:
    - cron: "0 2 * * 1-5" # 工作日凌晨 2 点（UTC）
    - cron: "0 0 * * 0" # 每周日零点


# cron 使用 UTC 时间，国内换算 +8 小时
```

### 手动触发

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "部署环境"
        required: true
        default: "staging"
        type: choice
        options: [dev, staging, prod]
      version:
        description: "版本号"
        required: false
        type: string
      dry-run:
        description: "演练模式"
        default: false
        type: boolean

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "部署到 ${{ inputs.environment }}，版本 ${{ inputs.version }}"
```

### 其他常用事件

```yaml
on:
  release:
    types: [published] # 发布 Release 时触发

  issues:
    types: [opened, labeled]

  workflow_call: # 被其他 Workflow 调用（可复用 Workflow）
    inputs:
      image-tag:
        required: true
        type: string
    secrets:
      deploy-token:
        required: true

  repository_dispatch: # 外部 API 触发
    types: [deploy-event]
```

---

## 内置上下文与变量

### 常用上下文

```yaml
steps:
  - run: |
      # github 上下文
      echo "仓库：${{ github.repository }}"         # org/repo
      echo "分支：${{ github.ref_name }}"
      echo "SHA：${{ github.sha }}"
      echo "短SHA：${{ github.sha }}" | cut -c1-8
      echo "事件：${{ github.event_name }}"
      echo "Actor：${{ github.actor }}"             # 触发者用户名
      echo "Run ID：${{ github.run_id }}"
      echo "Run Number：${{ github.run_number }}"

      # runner 上下文
      echo "OS：${{ runner.os }}"
      echo "临时目录：${{ runner.temp }}"

      # env 上下文
      echo "APP_NAME：${{ env.APP_NAME }}"
```

### 常用预定义变量

| 变量                 | 说明                        |
| -------------------- | --------------------------- |
| `$GITHUB_SHA`        | 完整 Commit SHA             |
| `$GITHUB_REF`        | 完整 ref（refs/heads/main） |
| `$GITHUB_REF_NAME`   | 分支或 Tag 名               |
| `$GITHUB_REPOSITORY` | 仓库名（owner/repo）        |
| `$GITHUB_WORKSPACE`  | 工作目录路径                |
| `$GITHUB_RUN_ID`     | Workflow 运行 ID            |
| `$GITHUB_RUN_NUMBER` | Workflow 运行编号           |
| `$GITHUB_ACTOR`      | 触发用户                    |
| `$GITHUB_EVENT_NAME` | 触发事件名                  |
| `$GITHUB_OUTPUT`     | 输出变量文件路径            |
| `$GITHUB_ENV`        | 环境变量文件路径            |
| `$RUNNER_TEMP`       | Runner 临时目录             |

### 在 Step 间传递输出

```yaml
steps:
  - name: 设置版本号
    id: version
    run: |
      VERSION=$(cat package.json | jq -r '.version')
      echo "version=$VERSION" >> $GITHUB_OUTPUT   # 写入输出

  - name: 使用版本号
    run: echo "版本是 ${{ steps.version.outputs.version }}"

  - name: 设置环境变量
    run: echo "IMAGE_TAG=v${{ steps.version.outputs.version }}" >> $GITHUB_ENV

  - name: 使用环境变量
    run: echo "$IMAGE_TAG"
```

---

## Job 配置详解

### needs（Job 依赖）

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: mvn package

  test:
    needs: build # 等待 build 完成后运行
    runs-on: ubuntu-latest
    steps:
      - run: mvn test

  deploy:
    needs: [build, test] # 等待多个 Job
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### 条件执行（if）

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    # Job 级别条件
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: 只在成功时部署
        if: success()
        run: ./deploy.sh

      - name: 失败时通知
        if: failure()
        run: ./notify-failure.sh

      - name: 始终运行（清理）
        if: always()
        run: ./cleanup.sh

      - name: 条件表达式
        if: contains(github.event.head_commit.message, '[deploy]')
        run: ./deploy.sh
```

**常用条件函数：**

| 函数                      | 说明             |
| ------------------------- | ---------------- |
| `success()`               | 前面步骤全部成功 |
| `failure()`               | 前面有步骤失败   |
| `always()`                | 始终执行         |
| `cancelled()`             | 工作流被取消     |
| `contains(str, substr)`   | 字符串包含       |
| `startsWith(str, prefix)` | 字符串前缀       |
| `endsWith(str, suffix)`   | 字符串后缀       |
| `format(str, ...)`        | 字符串格式化     |
| `join(array, sep)`        | 数组连接         |
| `toJSON(value)`           | 转 JSON          |
| `fromJSON(str)`           | 解析 JSON        |
| `hashFiles(pattern)`      | 计算文件哈希     |

### matrix（矩阵策略）

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false # 某个组合失败不取消其他
      max-parallel: 4 # 最多并行 4 个
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ["18", "20", "22"]
        include: # 额外添加组合
          - os: ubuntu-latest
            node: "20"
            experimental: true
        exclude: # 排除某个组合
          - os: windows-latest
            node: "18"

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

### environment（部署环境保护）

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com # 显示在 GitHub Deployments 页面
    steps:
      - run: ./deploy.sh prod
```

**在 GitHub 上配置 Environment 保护规则：**

- Settings → Environments → New environment
- Required reviewers：指定必须审批的人或团队
- Wait timer：部署前等待时间（分钟）
- Deployment branches：限定只有特定分支可以部署

### concurrency（并发控制）

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true # 新推送自动取消旧的运行

# Job 级别并发控制
jobs:
  deploy:
    concurrency:
      group: deploy-${{ inputs.environment }}
      cancel-in-progress: false # 部署不取消，排队等待
```

### timeout 与 retry

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30 # Job 超时

    steps:
      - name: 运行测试
        timeout-minutes: 15 # Step 超时
        continue-on-error: true # 步骤失败不终止 Job
        run: npm test

      - name: 重试步骤（借助 Action）
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 5
          max_attempts: 3
          command: ./flaky-test.sh
```

---

## 常用 Actions

### 代码检出与工具安装

```yaml
steps:
  # 检出代码
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0 # 获取完整历史（SonarQube 需要）
      ref: ${{ github.head_ref }} # PR 时检出源分支

  # Java
  - uses: actions/setup-java@v4
    with:
      java-version: "17"
      distribution: "temurin" # temurin / corretto / zulu / microsoft
      cache: "maven" # 自动缓存 Maven 依赖

  # Node.js
  - uses: actions/setup-node@v4
    with:
      node-version: "20"
      cache: "npm" # 自动缓存 npm

  # Python
  - uses: actions/setup-python@v5
    with:
      python-version: "3.12"
      cache: "pip"

  # Go
  - uses: actions/setup-go@v5
    with:
      go-version: "1.22"
      cache: true
```

### 缓存

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.m2/repository
      ~/.gradle/caches
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

### 产物上传/下载

```yaml
# 上传
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: |
      target/*.jar
      dist/
    retention-days: 7
    if-no-files-found: error # error | warn | ignore

# 下载（在另一个 Job 中）
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: ./artifacts
```

### Docker 构建与推送

```yaml
- name: Set up QEMU（多平台构建）
  uses: docker/setup-qemu-action@v3

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Extract metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/${{ github.repository }}
    tags: |
      type=ref,event=branch
      type=ref,event=pr
      type=semver,pattern={{version}}
      type=sha,prefix=sha-

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: ${{ github.event_name != 'pull_request' }}
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha # 使用 GitHub Actions 缓存层
    cache-to: type=gha,mode=max
```

### GitHub Release

```yaml
- name: Create Release
  uses: softprops/action-gh-release@v2
  with:
    tag_name: ${{ github.ref_name }}
    name: Release ${{ github.ref_name }}
    body_path: CHANGELOG.md
    files: |
      dist/*.zip
      target/*.jar
    draft: false
    prerelease: ${{ contains(github.ref_name, '-rc') }}
```

### 代码质量

```yaml
# SonarCloud
- name: SonarCloud Scan
  uses: SonarSource/sonarcloud-github-action@master
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

# CodeQL（GitHub 原生代码扫描）
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: java, javascript

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v3
```

---

## Secrets 与变量管理

### 分级管理

| 级别                | 路径                               | 作用域         |
| ------------------- | ---------------------------------- | -------------- |
| Repository Secret   | Settings → Secrets → Actions       | 当前仓库       |
| Repository Variable | Settings → Variables → Actions     | 当前仓库       |
| Environment Secret  | Settings → Environments → 指定环境 | 特定环境的 Job |
| Organization Secret | Org Settings → Secrets             | 组织内多个仓库 |

```yaml
env:
  # 引用 Secret（加密，日志自动打码）
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

  # 引用 Variable（明文）
  API_URL: ${{ vars.API_URL }}

  # GitHub 自动提供的 Token（当前仓库权限）
  GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### GITHUB_TOKEN 权限

```yaml
# 工作流级别权限（最小权限原则）
permissions:
  contents: read
  packages: write # 推送到 GHCR 需要
  pull-requests: write # 评论 PR 需要
  id-token: write # OIDC 认证需要

# Job 级别覆盖
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
```

### OIDC 无密钥认证（推荐，代替长期凭据）

```yaml
# 以 AWS 为例，无需存储 Access Key
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: cn-north-1
      - run: aws s3 sync dist/ s3://my-bucket/
```

---

## 可复用工作流（Reusable Workflows）

### 定义可复用工作流

```yaml
# .github/workflows/deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      deploy-token:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy
        env:
          TOKEN: ${{ secrets.deploy-token }}
        run: |
          echo "部署 ${{ inputs.image-tag }} 到 ${{ inputs.environment }}"
          ./deploy.sh ${{ inputs.environment }} ${{ inputs.image-tag }}
```

### 调用可复用工作流

```yaml
# .github/workflows/ci.yml
jobs:
  build:
    uses: ./.github/workflows/build.yml # 本仓库

  deploy-staging:
    needs: build
    uses: my-org/shared-workflows/.github/workflows/deploy.yml@main # 其他仓库
    with:
      environment: staging
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets:
      deploy-token: ${{ secrets.STAGING_DEPLOY_TOKEN }}

  deploy-prod:
    needs: deploy-staging
    uses: my-org/shared-workflows/.github/workflows/deploy.yml@main
    with:
      environment: production
      image-tag: ${{ needs.build.outputs.image-tag }}
    secrets: inherit # 继承调用方所有 secrets
```

---

## Composite Action（自定义复合 Action）

将多个步骤封装为一个 Action，在多个工作流中复用。

```yaml
# .github/actions/setup-and-cache/action.yml
name: "Setup Node and Cache"
description: "安装 Node.js 并恢复缓存"

inputs:
  node-version:
    description: "Node.js 版本"
    required: false
    default: "20"

outputs:
  cache-hit:
    description: "是否命中缓存"
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: "composite"
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - id: cache
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

    - if: steps.cache.outputs.cache-hit != 'true'
      run: npm ci
      shell: bash
```

```yaml
# 在工作流中使用
steps:
  - uses: ./.github/actions/setup-and-cache
    with:
      node-version: "20"
```

---

## Self-hosted Runner

### 安装 Runner

```shell
# 在 GitHub 仓库：Settings → Actions → Runners → New self-hosted runner
# 按页面上的命令操作，以 Linux 为例：

mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.x.x.tar.gz -L https://github.com/actions/runner/releases/download/v2.x.x/actions-runner-linux-x64-2.x.x.tar.gz
tar xzf ./actions-runner-linux-x64-2.x.x.tar.gz

# 配置（按页面上的 token）
./config.sh --url https://github.com/your-org/your-repo --token YOUR_TOKEN --labels "self-hosted,linux,docker"

# 安装为服务
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

### 使用 Self-hosted Runner

```yaml
jobs:
  build:
    runs-on: self-hosted # 使用任意 self-hosted runner

  deploy:
    runs-on: [self-hosted, linux, docker] # 匹配标签
```

### Docker 中运行 Runner（推荐）

```yaml
# docker-compose.yml
version: "3.8"
services:
  runner:
    image: myoung34/github-runner:latest
    restart: unless-stopped
    environment:
      RUNNER_SCOPE: repo
      REPO_URL: https://github.com/your-org/your-repo
      RUNNER_TOKEN: YOUR_TOKEN
      RUNNER_NAME: docker-runner-1
      LABELS: docker,linux
      RUNNER_WORKDIR: /tmp/runner/work
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/runner:/tmp/runner
```

---

## 完整示例

### Java Spring Boot CI/CD

```yaml
name: Java CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [published]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  JAVA_VERSION: "17"

jobs:
  # ── Build & Test ──────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      checks: write # 发布测试报告

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Run Tests
        run: mvn verify

      - name: Publish Test Report
        uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: "**/target/surefire-reports/*.xml"

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          files: target/site/jacoco/jacoco.xml
          token: ${{ secrets.CODECOV_TOKEN }}

  # ── Build Docker Image ────────────────────────────
  build-image:
    name: Build Image
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Build JAR
        run: mvn package -DskipTests

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-,format=short

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── Deploy to Dev ─────────────────────────────────
  deploy-dev:
    name: Deploy Dev
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://dev.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Dev
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USER }}
          key: ${{ secrets.DEV_SSH_KEY }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }}
            docker stop app || true
            docker rm   app || true
            docker run -d --name app --restart unless-stopped \
              -p 8080:8080 \
              ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-image.outputs.image-tag }}

  # ── Deploy to Production ──────────────────────────
  deploy-prod:
    name: Deploy Production
    needs: build-image
    runs-on: ubuntu-latest
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://app.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: cn-north-1

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster prod-cluster \
            --service app-service \
            --force-new-deployment

      - name: Wait for stable
        run: |
          aws ecs wait services-stable \
            --cluster prod-cluster \
            --services app-service

  # ── Notify ────────────────────────────────────────
  notify:
    name: Notify
    needs: [deploy-dev, deploy-prod]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Send notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "${{ needs.deploy-prod.result == 'success' && '✅' || '❌' }} ${{ github.repository }} - ${{ github.workflow }} ${{ needs.deploy-prod.result }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Node.js 前端 CI/CD

```yaml
name: Frontend CI/CD

on:
  push:
    branches: [main]
    paths: ["src/**", "package*.json", "vite.config.*"]
  pull_request:
    branches: [main]

jobs:
  quality:
    name: Lint & Test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run type-check

      - name: Unit tests
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
          retention-days: 3

  build:
    name: Build
    needs: quality
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    strategy:
      matrix:
        env: [staging, production]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: Build (${{ matrix.env }})
        run: npm run build
        env:
          VITE_APP_ENV: ${{ matrix.env }}
          VITE_API_URL: ${{ vars[format('API_URL_{0}', upper(matrix.env))] }}

      - uses: actions/upload-artifact@v4
        with:
          name: dist-${{ matrix.env }}
          path: dist/
          retention-days: 3

  deploy:
    name: Deploy to ${{ matrix.env }}
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - env: staging
            bucket: s3://staging-bucket
            distribution: STAGING_CF_ID
          - env: production
            bucket: s3://prod-bucket
            distribution: PROD_CF_ID
    environment:
      name: ${{ matrix.env }}
      url: ${{ matrix.env == 'production' && 'https://app.example.com' || 'https://staging.example.com' }}

    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist-${{ matrix.env }}
          path: dist/

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', upper(matrix.env))] }}
          aws-region: cn-north-1

      - name: Deploy to S3
        run: aws s3 sync dist/ ${{ matrix.bucket }} --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets[matrix.distribution] }} \
            --paths "/*"
```

---

## 常用工作流片段

### PR 自动标签

```yaml
name: PR Labeler
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  label:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/labeler@v5
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
          configuration-path: .github/labeler.yml
```

```yaml
# .github/labeler.yml
frontend:
  - changed-files:
      - any-glob-to-any-file: ["src/frontend/**", "*.css", "*.scss"]
backend:
  - changed-files:
      - any-glob-to-any-file: ["src/backend/**", "pom.xml"]
```

### 自动合并 Dependabot PR

```yaml
name: Auto merge Dependabot
on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'
    steps:
      - uses: dependabot/fetch-metadata@v2
        id: metadata

      - name: Auto merge patch and minor updates
        if: steps.metadata.outputs.update-type != 'version-update:semver-major'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 代码检查（PR 评论）

```yaml
name: Code Review Bot
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run ESLint and comment
        uses: reviewdog/action-eslint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review # 在 PR 行级别发表评论
          eslint_flags: "src/"
```

---

## 最佳实践

1. **锁定 Action 版本**：使用 commit SHA 而非 tag（如 `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`），防止供应链攻击。版本 tag 可能被覆盖，SHA 不可变。

2. **最小 GITHUB_TOKEN 权限**：工作流顶层设置 `permissions: read-all`，各 Job 按需授权，遵循最小权限原则。

3. **不在工作流中硬编码密钥**：所有敏感信息通过 `${{ secrets.XXX }}` 引用，优先使用 OIDC 代替长期凭据。

4. **缓存加速**：使用 `actions/cache` 或 setup-\* Action 内置缓存（Maven、npm、pip），缓存 key 以锁文件哈希为准。

5. **并发控制**：生产部署配置 `concurrency` 防止并发部署竞争，PR 场景设 `cancel-in-progress: true` 节省资源。

6. **PR 快速反馈**：lint、单测等轻量检查在 PR 阶段运行，完整构建和部署只在 push main/release 时触发。

7. **矩阵测试**：用 `matrix` 覆盖多平台 / 多版本，`fail-fast: false` 确保不因一个组合失败而取消所有。

8. **可复用工作流**：公共的 build / deploy 逻辑抽取为 Reusable Workflow 或 Composite Action，在组织内共享。

9. **Environment 保护**：生产环境配置 Required Reviewers，确保人工审批后才部署，同时限制只有特定分支可以访问。

10. **构建产物版本化**：使用 `docker/metadata-action` 自动生成语义化镜像 Tag，保证可追溯和可回滚。

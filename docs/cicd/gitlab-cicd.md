# GitLab CI/CD

GitLab CI/CD 是 GitLab 内置的持续集成 / 持续交付系统，通过项目根目录的 `.gitlab-ci.yml` 文件定义流水线，无需额外安装服务器，推送代码即可自动触发。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Pipeline** | 一次完整的 CI/CD 执行过程，由多个 Stage 组成 |
| **Stage** | 阶段，同一 Stage 内的 Job 并行执行，Stage 之间串行 |
| **Job** | 最小执行单元，定义具体执行什么命令 |
| **Runner** | 执行 Job 的代理程序，分 Shared / Group / Project Runner |
| **Artifact** | Job 产物，可在后续 Job 中下载使用 |
| **Cache** | 跨 Job / Pipeline 复用的缓存（如 node_modules） |
| **Environment** | 部署目标环境，如 dev / staging / production |
| **Variable** | 变量，可在 UI 中配置敏感信息，Job 中通过 `$VAR` 引用 |
| **Trigger** | 触发器，支持 Push、MR、Tag、Schedule、API 等 |
| **Include** | 引入外部 YAML 片段，实现配置复用 |

---

## .gitlab-ci.yml 基本结构

```yaml
# 定义 Stage 执行顺序（同 Stage 并行，Stage 间串行）
stages:
  - build
  - test
  - scan
  - package
  - deploy

# 全局变量
variables:
  APP_NAME: my-app
  DOCKER_REGISTRY: registry.gitlab.com/$CI_PROJECT_PATH

# 全局默认配置（所有 Job 继承）
default:
  image: ubuntu:22.04
  tags:
    - docker
  before_script:
    - echo "开始执行：$CI_JOB_NAME"
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure

# ── Job 定义 ──────────────────────────────────────────
build:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 hour

unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/**/*.xml

deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://app.example.com
  when: manual                   # 手动触发
  only:
    - main
```

---

## 内置变量（Predefined Variables）

| 变量 | 说明 |
|------|------|
| `$CI_PIPELINE_ID` | Pipeline 唯一 ID |
| `$CI_PIPELINE_IID` | Pipeline 项目内编号 |
| `$CI_JOB_ID` | Job 唯一 ID |
| `$CI_JOB_NAME` | Job 名称 |
| `$CI_JOB_STAGE` | Job 所在 Stage |
| `$CI_COMMIT_SHA` | Commit SHA（完整） |
| `$CI_COMMIT_SHORT_SHA` | Commit SHA（前 8 位） |
| `$CI_COMMIT_BRANCH` | 分支名 |
| `$CI_COMMIT_TAG` | Tag 名（Tag 触发时） |
| `$CI_COMMIT_MESSAGE` | Commit 信息 |
| `$CI_DEFAULT_BRANCH` | 默认分支（通常是 main） |
| `$CI_PROJECT_NAME` | 项目名 |
| `$CI_PROJECT_PATH` | 项目路径（group/project） |
| `$CI_PROJECT_URL` | 项目 URL |
| `$CI_REGISTRY` | GitLab Container Registry 地址 |
| `$CI_REGISTRY_IMAGE` | 当前项目的 Registry 镜像地址 |
| `$CI_REGISTRY_USER` | Registry 登录用户名 |
| `$CI_REGISTRY_PASSWORD` | Registry 登录密码 |
| `$CI_MERGE_REQUEST_IID` | MR 编号（MR Pipeline 中） |
| `$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` | MR 源分支 |
| `$CI_ENVIRONMENT_NAME` | 部署环境名 |
| `$CI_RUNNER_DESCRIPTION` | Runner 描述 |
| `$GITLAB_USER_NAME` | 触发构建的用户名 |

---

## Job 配置详解

### image 与 services

```yaml
build:
  image: node:20-alpine          # 使用指定镜像运行 Job

test-with-db:
  image: python:3.12
  services:
    - name: postgres:15           # 启动附加服务容器
      alias: db                   # 通过 hostname "db" 访问
    - name: redis:7
      alias: cache
  variables:
    POSTGRES_DB:       test_db
    POSTGRES_USER:     runner
    POSTGRES_PASSWORD: secret
    DATABASE_URL:      postgresql://runner:secret@db/test_db
  script:
    - pip install -r requirements.txt
    - pytest
```

### script、before_script、after_script

```yaml
deploy:
  before_script:
    - apt-get update -qq && apt-get install -y curl
    - curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    - chmod +x kubectl && mv kubectl /usr/local/bin/
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$IMAGE_TAG
  after_script:
    - echo "部署完成，清理临时文件"
    - rm -f /tmp/kubeconfig
```

### artifacts（产物）

```yaml
build:
  script:
    - mvn clean package
  artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"   # 产物名称
    paths:
      - target/*.jar
      - dist/
    exclude:
      - target/**/*.tmp
    expire_in: 7 days              # 过期时间（不设置则根据实例配置）
    when: on_success               # on_success | on_failure | always
    reports:
      junit:    target/surefire-reports/**/*.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      sast:     gl-sast-report.json
      dast:     gl-dast-report.json
```

### cache（缓存）

```yaml
# 全局缓存配置
cache:
  key:
    files:
      - package-lock.json          # 以锁文件内容为 key（内容变化才失效）
  paths:
    - node_modules/
    - .npm/
  policy: pull-push                # pull-push（默认）| pull | push

# Job 级别覆盖缓存
build:
  cache:
    key: "$CI_COMMIT_REF_SLUG"     # 以分支名为 key
    paths:
      - .gradle/
    policy: pull                   # 只拉不推（只读缓存，加速）
```

### rules（触发规则）— 推荐方式

```yaml
deploy-prod:
  script: ./deploy.sh prod
  rules:
    # 在 main 分支且不是 MR 时运行（手动确认）
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE != "merge_request_event"'
      when: manual
    # 在 Tag 上自动运行
    - if: '$CI_COMMIT_TAG'
      when: on_success
    # 其他情况不运行
    - when: never

# 文件变更触发
frontend-build:
  script: npm run build
  rules:
    - changes:
        - src/frontend/**/*
        - package*.json
      when: on_success
    - when: never

# 变量条件
integration-test:
  script: npm run test:e2e
  rules:
    - if: '$RUN_E2E == "true"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### only / except（旧写法，了解即可）

```yaml
# only：只在满足条件时运行
deploy:
  only:
    - main
    - /^release\/.*/    # 正则匹配 release/* 分支
    - tags

# except：排除条件
test:
  except:
    - schedules
    - triggers
```

### when（执行时机）

| 值 | 说明 |
|----|------|
| `on_success` | 前置 Job 全部成功（默认） |
| `on_failure` | 有 Job 失败时运行（常用于通知） |
| `always` | 无论成功失败都运行 |
| `manual` | 手动触发（Pipeline 暂停等待点击） |
| `delayed` | 延迟执行（配合 `start_in`） |
| `never` | 不运行 |

```yaml
notify-on-failure:
  stage: .post
  script: ./notify.sh "构建失败"
  when: on_failure

delayed-deploy:
  stage: deploy
  script: ./deploy.sh canary
  when: delayed
  start_in: 30 minutes
```

### needs（DAG 依赖，打破 Stage 限制）

```yaml
# 不等待同 Stage 其他 Job，直接依赖指定 Job 完成后运行
# 实现有向无环图（DAG）调度，大幅缩短 Pipeline 时长

build-backend:
  stage: build
  script: mvn package

build-frontend:
  stage: build
  script: npm run build

# 不等 test Stage 所有 Job，只要 build-backend 完成即可运行
test-backend:
  stage: test
  needs: [build-backend]
  script: mvn test

# 跨 Stage 依赖，直接在 build-frontend 完成后运行（跳过 test Stage 等待）
deploy-frontend:
  stage: deploy
  needs:
    - job: build-frontend
      artifacts: true             # 下载该 Job 的产物
  script: aws s3 sync dist/ s3://bucket/
```

### parallel（矩阵并行）

```yaml
# 同一 Job 以不同参数并行运行
test:
  stage: test
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.10", "3.11", "3.12"]
        DB: ["postgres", "mysql"]
  image: python:$PYTHON_VERSION
  script:
    - pip install -r requirements.txt
    - pytest --db=$DB

# 简单并行（拆分测试）
test:
  parallel: 5               # 自动将测试分成 5 份并行跑
  script: pytest --splits $CI_NODE_TOTAL --group $CI_NODE_INDEX
```

### environment（部署环境）

```yaml
deploy-staging:
  stage: deploy
  script: ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop-staging          # 关联停止 Job

stop-staging:
  stage: deploy
  script: ./teardown.sh staging
  environment:
    name: staging
    action: stop
  when: manual
```

---

## 变量管理

### 定义变量的优先级（高→低）

1. 触发 Pipeline 时传入的变量（API / Trigger）
2. Pipeline 级别变量（`.gitlab-ci.yml` 中的 `variables`）
3. Project / Group / Instance 级别变量（UI 配置）
4. `.gitlab-ci.yml` 中 `default.variables`

### 在 UI 中配置变量

**Settings → CI/CD → Variables**

| 类型 | 说明 |
|------|------|
| Variable | 普通键值对 |
| File | 将内容写入临时文件，变量值为文件路径 |
| 勾选 Masked | 日志中自动打码 |
| 勾选 Protected | 只在受保护分支 / Tag 的 Pipeline 中可用 |

```yaml
# 引用变量
deploy:
  script:
    - echo $APP_ENV
    - kubectl --server=$K8S_SERVER --token=$K8S_TOKEN apply -f k8s/
    # File 类型变量：$KUBECONFIG 是文件路径
    - kubectl --kubeconfig=$KUBECONFIG get pods
```

### 组变量与继承

组级别（Group → Settings → CI/CD → Variables）定义的变量会自动被所有子项目和子组继承，适合在多项目中共享 Registry 地址、公共 Token 等配置。

---

## 触发器（Triggers）

### Push / MR / Tag 触发（默认）

```yaml
# 只在 MR 时运行（MR Pipeline）
lint:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# 只在 Tag 时运行
release:
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
```

### 定时触发（Scheduled Pipeline）

**CI/CD → Schedules → New schedule**

```yaml
# 定时 Pipeline 中才运行的 Job
nightly-build:
  script: ./full-regression-test.sh
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
```

### API 触发

```shell
# 触发指定 ref 的 Pipeline
curl -X POST \
  -F "token=YOUR_TRIGGER_TOKEN" \
  -F "ref=main" \
  -F "variables[ENV]=production" \
  "https://gitlab.example.com/api/v4/projects/123/trigger/pipeline"
```

### 上游 / 下游 Pipeline（Multi-project Pipeline）

```yaml
# 在当前 Pipeline 触发另一个项目的 Pipeline
trigger-deploy:
  stage: deploy
  trigger:
    project: devops/deploy-repo
    branch:  main
    strategy: depend              # 等待子 Pipeline 完成，状态同步
  variables:
    IMAGE_TAG: $CI_COMMIT_SHORT_SHA
```

---

## GitLab Container Registry

GitLab 内置容器镜像仓库，无需额外配置。

```yaml
variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE .
    - docker push $IMAGE
    # 同时打 latest
    - docker tag $IMAGE $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
```

### 使用 Kaniko（无需特权模式）

```yaml
build-image:
  stage: package
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  script:
    - /kaniko/executor
      --context=$CI_PROJECT_DIR
      --dockerfile=$CI_PROJECT_DIR/Dockerfile
      --destination=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
      --destination=$CI_REGISTRY_IMAGE:latest
      --cache=true
```

---

## include（配置复用）

```yaml
# 引入本仓库其他文件
include:
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/test.yml'

# 引入其他项目的模板
  - project: 'devops/ci-templates'
    ref:     main
    file:    '/templates/docker-build.yml'

# 引入 GitLab 官方模板
  - template: Security/SAST.gitlab-ci.yml
  - template: Code-Quality.gitlab-ci.yml

# 引入远程 URL
  - remote: 'https://raw.githubusercontent.com/example/ci/main/template.yml'
```

### 模板 Job（使用 `&` 锚点和 `extends`）

```yaml
# 定义模板（以 . 开头的 Job 不会被执行）
.deploy-template:
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl
    - curl -LO https://dl.k8s.io/release/.../kubectl
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$IMAGE_TAG -n $NAMESPACE
  environment:
    url: https://$APP_NAME.$NAMESPACE.example.com

# 继承模板
deploy-staging:
  extends: .deploy-template
  variables:
    NAMESPACE: staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-prod:
  extends: .deploy-template
  variables:
    NAMESPACE: production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## GitLab Runner

### 安装 Runner（Linux）

```shell
# 添加仓库
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner

# 注册 Runner（交互式）
sudo gitlab-runner register

# 注册 Runner（非交互式）
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com/" \
  --registration-token "YOUR_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "docker-runner" \
  --tag-list "docker,linux" \
  --run-untagged="true" \
  --locked="false"

# 启动服务
sudo gitlab-runner start
sudo gitlab-runner status
```

### 安装 Runner（Docker）

```shell
docker run -d \
  --name gitlab-runner \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v gitlab-runner-config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest

# 注册
docker exec -it gitlab-runner gitlab-runner register
```

### Runner Executor 类型

| Executor | 说明 | 适用场景 |
|----------|------|---------|
| `docker` | 每个 Job 在独立容器运行 | 推荐，隔离性好 |
| `shell` | 直接在 Runner 机器上运行 | 简单场景 |
| `kubernetes` | 在 K8s Pod 中运行 | 云原生环境 |
| `docker+machine` | 自动伸缩 Docker 机器 | 大规模构建 |
| `virtualbox` | 在虚拟机中运行 | 需要完整 OS |

### Runner 配置（/etc/gitlab-runner/config.toml）

```toml
concurrent = 4              # 最大并发 Job 数

[[runners]]
  name = "docker-runner"
  url = "https://gitlab.example.com"
  token = "YOUR_TOKEN"
  executor = "docker"
  [runners.docker]
    image = "alpine:latest"
    privileged = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    pull_policy = ["if-not-present"]   # 优先使用本地镜像
  [runners.cache]
    Type = "s3"
    [runners.cache.s3]
      ServerAddress = "minio.example.com"
      AccessKey = "ACCESS_KEY"
      SecretKey = "SECRET_KEY"
      BucketName = "gitlab-runner-cache"
```

---

## 安全扫描（DevSecOps）

GitLab 内置多种安全扫描，Ultimate 版本全部免费，CE/EE 版本可使用模板。

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml           # 静态代码安全分析
  - template: Security/Secret-Detection.gitlab-ci.yml # 敏感信息检测
  - template: Security/Dependency-Scanning.gitlab-ci.yml # 依赖漏洞扫描
  - template: Security/Container-Scanning.gitlab-ci.yml  # 容器镜像扫描
  - template: Security/DAST.gitlab-ci.yml           # 动态应用安全测试

variables:
  SAST_EXCLUDED_PATHS: "spec,test,tests,tmp"
  DS_EXCLUDED_PATHS:   "node_modules"
```

---

## 完整示例

### Java Spring Boot 项目

```yaml
stages:
  - build
  - test
  - scan
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  IMAGE_TAG:  "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"

default:
  image: maven:3.9-eclipse-temurin-17
  cache:
    key:
      files: [pom.xml]
    paths: [.m2/repository/]

# ── Build ────────────────────────────────────────────
compile:
  stage: build
  script:
    - mvn compile -DskipTests
  artifacts:
    paths: [target/classes/]
    expire_in: 1 hour

# ── Test ─────────────────────────────────────────────
unit-test:
  stage: test
  script:
    - mvn test
  artifacts:
    when: always
    reports:
      junit:    target/surefire-reports/**/*.xml
    paths:    [target/jacoco.exec]
    expire_in: 1 day

code-quality:
  stage: test
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_USER_HOME: "$CI_PROJECT_DIR/.sonar"
    GIT_DEPTH:       "0"
  cache:
    key: sonar-$CI_COMMIT_REF_SLUG
    paths: [.sonar/cache]
  script:
    - sonar-scanner
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ── Package ──────────────────────────────────────────
package:
  stage: package
  script:
    - mvn package -DskipTests
  artifacts:
    paths: [target/*.jar]
    expire_in: 7 days

build-image:
  stage: package
  image: docker:24
  services: [docker:24-dind]
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
        docker push $CI_REGISTRY_IMAGE:latest
      fi
  needs: [package]

# ── Deploy ───────────────────────────────────────────
.deploy-base:
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server=$K8S_SERVER --certificate-authority=$K8S_CA
    - kubectl config set-credentials ci --token=$K8S_TOKEN
    - kubectl config set-context ci --cluster=k8s --user=ci --namespace=$NAMESPACE
    - kubectl config use-context ci
    - sed -i "s|IMAGE_TAG|$IMAGE_TAG|g" k8s/deployment.yaml
    - kubectl apply -f k8s/
    - kubectl rollout status deployment/$APP_NAME -n $NAMESPACE --timeout=120s

deploy-dev:
  extends: .deploy-base
  stage: deploy
  variables:
    NAMESPACE: dev
    APP_NAME:  my-app
  environment:
    name: development
    url:  https://dev.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-staging:
  extends: .deploy-base
  stage: deploy
  variables:
    NAMESPACE: staging
    APP_NAME:  my-app
  environment:
    name: staging
    url:  https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^release\//'

deploy-prod:
  extends: .deploy-base
  stage: deploy
  variables:
    NAMESPACE: production
    APP_NAME:  my-app
  environment:
    name: production
    url:  https://app.example.com
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
      when: on_success
```

### Node.js 前端项目

```yaml
stages:
  - install
  - lint-test
  - build
  - deploy

variables:
  NODE_ENV: production

default:
  image: node:20-alpine
  cache:
    key:
      files: [package-lock.json]
    paths: [node_modules/, .npm/]

install:
  stage: install
  script:
    - npm ci --cache .npm --prefer-offline
  artifacts:
    paths: [node_modules/]
    expire_in: 1 hour

lint:
  stage: lint-test
  needs: [install]
  script:
    - npm run lint
    - npm run type-check

test:
  stage: lint-test
  needs: [install]
  script:
    - npm run test -- --coverage --watchAll=false
  artifacts:
    when: always
    reports:
      junit:    junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'

build:
  stage: build
  needs: [lint, test]
  script:
    - npm run build
  artifacts:
    paths: [dist/]
    expire_in: 7 days
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never

pages:                                   # GitLab Pages 自动部署
  stage: deploy
  needs: [build]
  script:
    - mv dist public
  artifacts:
    paths: [public]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## 最佳实践

1. **rules 替代 only/except**：`rules` 语法更灵活、可读性更好，新项目应统一使用 `rules`。
2. **合理使用 needs**：通过 DAG 依赖消除不必要的等待，尤其是并行的 build + test 阶段。
3. **缓存策略**：用锁文件（`package-lock.json` / `pom.xml`）作为缓存 key，保证缓存精确失效。
4. **产物精简**：只将后续 Job 需要的文件声明为 artifact，减少传输开销。设置合理的 `expire_in`。
5. **敏感变量**：密码、Token 一律在 GitLab UI 中配置为 Masked + Protected 变量，禁止写入 `.gitlab-ci.yml`。
6. **使用 include 模板**：公共的 build / deploy 逻辑抽取到共享模板库，各项目通过 `include` 引入。
7. **MR Pipeline 快速反馈**：lint、单测等轻量 Job 在 MR Pipeline 中运行，完整流程只在主干分支触发。
8. **手动审批生产部署**：`when: manual` + 配置 Protected Environment（只允许特定角色操作）。
9. **限制并发**：在 Runner 配置中设置合理的 `concurrent`，防止资源耗尽。
10. **Job 超时**：为长 Job 设置 `timeout` 防止僵尸任务，默认超时为 60 分钟。

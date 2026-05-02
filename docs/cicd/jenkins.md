# Jenkins

Jenkins 是一款开源的持续集成 / 持续交付（CI/CD）自动化服务器，用 Java 编写，通过插件扩展可以对接几乎任何构建、测试、部署工具。它是目前使用最广泛的 CI/CD 平台之一。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Job / Project** | 一个可执行的任务单元，定义了要做什么 |
| **Build** | Job 的一次执行实例 |
| **Pipeline** | 用代码描述的完整 CI/CD 流程（推荐方式） |
| **Stage** | Pipeline 中的一个阶段，如 Build、Test、Deploy |
| **Step** | Stage 内的单个操作 |
| **Node / Agent** | 执行构建任务的机器（Master 或 Worker） |
| **Executor** | Node 上并发执行构建的槽位数量 |
| **Workspace** | 构建时的工作目录，存放源码和产物 |
| **Plugin** | 插件，Jenkins 功能的主要扩展方式（目前 1800+） |
| **Jenkinsfile** | 用代码定义 Pipeline 的文件，存放在代码仓库中 |
| **Credential** | 安全存储的凭据（密码、Token、SSH Key 等） |
| **Artifact** | 构建产物，如 JAR、APK、Docker 镜像 |
| **Trigger** | 触发构建的条件，如 Push、定时、上游 Job 完成 |

---

## 安装

### 方式一：Docker 安装（推荐）

```shell
# 创建数据卷目录
mkdir -p /var/jenkins_home
chown -R 1000:1000 /var/jenkins_home

# 启动 Jenkins（含 Docker-in-Docker 支持）
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /var/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk17

# 查看初始管理员密码
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

> 端口说明：`8080` 为 Web UI，`50000` 为 Agent（Inbound）连接端口。

### 方式二：Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Duser.timezone=Asia/Shanghai

volumes:
  jenkins_home:
```

```shell
docker compose up -d
docker compose logs -f jenkins
```

### 方式三：Linux 原生安装（Ubuntu/Debian）

```shell
# 添加 Jenkins 仓库
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# 安装 Java 和 Jenkins
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre jenkins

# 启动服务
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

# 查看初始密码
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 方式四：Linux 原生安装（CentOS/RHEL）

```shell
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

sudo dnf install -y java-17-openjdk jenkins
sudo systemctl enable --now jenkins

sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 初始化向导

1. 浏览器访问 `http://localhost:8080`
2. 输入初始管理员密码
3. 选择 **Install suggested plugins**（安装推荐插件）
4. 创建第一个管理员账号
5. 配置 Jenkins URL（保持默认即可）

---

## 基础配置

### 全局工具配置

路径：**Manage Jenkins → Tools**

```
# 配置 JDK
Name: JDK17
JAVA_HOME: /usr/lib/jvm/java-17-openjdk

# 配置 Maven
Name: Maven3
Install automatically: 勾选
Version: 3.9.x

# 配置 Node.js（需安装 NodeJS 插件）
Name: NodeJS20
Install automatically: 勾选
Version: 20.x
```

### 系统配置

路径：**Manage Jenkins → System**

```
# 重要配置项
Jenkins URL:          http://your-jenkins-host:8080/
System Admin e-mail:  admin@example.com
# of executors:       4（建议设为 CPU 核心数）
```

### 管理凭据

路径：**Manage Jenkins → Credentials → System → Global credentials**

| 凭据类型 | 用途 |
|---------|------|
| Username with password | Git 账号、Maven 仓库等 |
| SSH Username with private key | SSH 登录服务器、Git SSH |
| Secret text | API Token、密码字符串 |
| Secret file | 配置文件、证书文件 |
| Certificate | PKCS#12 证书 |

```
# 添加 Git SSH Key
Kind: SSH Username with private key
ID: github-ssh-key          ← 在 Pipeline 中通过此 ID 引用
Username: git
Private Key: 粘贴私钥内容
```

### 配置 Git

```shell
# 在 Jenkins 服务器上配置全局 Git 用户信息
git config --global user.name "Jenkins"
git config --global user.email "jenkins@example.com"
```

---

## Pipeline 基础

Pipeline 是 Jenkins 推荐的任务定义方式，将整个 CI/CD 流程写成代码（Pipeline as Code），存储在项目根目录的 `Jenkinsfile` 中，随代码版本管理。

### 两种语法风格

| 风格 | 特点 | 推荐程度 |
|------|------|---------|
| **Declarative**（声明式） | 结构固定、可读性强、有语法校验 | ★★★★★ 推荐 |
| **Scripted**（脚本式） | 完整 Groovy 语法、灵活但复杂 | 复杂场景使用 |

---

## Declarative Pipeline 详解

### 基本结构

```groovy
pipeline {
    agent any               // 在任意可用 Agent 上运行

    environment {           // 环境变量
        APP_NAME = 'my-app'
        VERSION  = '1.0.0'
    }

    options {               // 全局选项
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {              // 触发器
        cron('H 2 * * 1-5') // 工作日凌晨 2 点
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {                  // 构建后动作
        success {
            echo '构建成功！'
        }
        failure {
            mail to: 'team@example.com', subject: '构建失败', body: "${env.BUILD_URL}"
        }
        always {
            cleanWs()       // 清理工作目录
        }
    }
}
```

### agent 指令

```groovy
// 在任意可用节点运行
agent any

// 不分配 Agent（常用于顶层，各 Stage 单独指定）
agent none

// 指定标签的节点
agent { label 'linux && docker' }

// 在 Docker 容器中运行
agent {
    docker {
        image 'node:20-alpine'
        args  '-v /tmp:/tmp'
        reuseNode true     // 复用上层节点（节省切换开销）
    }
}

// 使用 Dockerfile 构建镜像
agent {
    dockerfile {
        filename 'ci/Dockerfile.build'
        dir      'ci'
        args     '--build-arg ENV=ci'
    }
}
```

### environment 指令

```groovy
environment {
    // 普通变量
    APP_ENV = 'production'

    // 引用凭据（自动注入为环境变量）
    GIT_TOKEN    = credentials('github-token')      // Secret text → 单变量
    DOCKER_CREDS = credentials('dockerhub-creds')   // user/pass → 生成 _USR / _PSW 两个变量
    SSH_KEY      = credentials('deploy-ssh-key')    // SSH Key → 文件路径
}

steps {
    sh 'echo $APP_ENV'
    sh 'echo $DOCKER_CREDS_USR'   // dockerhub 用户名
    sh 'echo $DOCKER_CREDS_PSW'   // dockerhub 密码（日志中自动打码）
    sh 'docker login -u $DOCKER_CREDS_USR -p $DOCKER_CREDS_PSW'
}
```

### when 条件

```groovy
stage('Deploy to Prod') {
    when {
        // 单个条件
        branch 'main'

        // 多个条件（AND 关系）
        allOf {
            branch 'main'
            not { changeRequest() }           // 不是 PR
            environment name: 'DEPLOY', value: 'true'
        }

        // 任一条件（OR 关系）
        anyOf {
            branch 'main'
            branch 'release/*'
        }

        // 表达式条件
        expression { return params.DEPLOY_ENABLED == true }

        // 文件变更
        changeset 'src/backend/**'

        // Tag
        tag 'v*'
    }
    steps { ... }
}
```

### 并行 Stage

```groovy
stage('Parallel Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'mvn test -pl unit-tests'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'mvn test -pl integration-tests'
            }
        }
        stage('E2E Tests') {
            agent { label 'selenium' }
            steps {
                sh 'npm run e2e'
            }
        }
    }
}
```

### 参数化构建

```groovy
parameters {
    string(name: 'BRANCH',     defaultValue: 'main',        description: '分支名')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false,   description: '跳过测试')
    choice(name: 'ENV',
           choices: ['dev', 'staging', 'prod'],             description: '部署环境')
    password(name: 'DB_PASS',  defaultValue: '',            description: '数据库密码')
    text(name: 'RELEASE_NOTES',defaultValue: '',            description: '发布说明')
    file(name: 'CONFIG_FILE',                               description: '上传配置文件')
}

// 使用参数
steps {
    sh "git checkout ${params.BRANCH}"
    script {
        if (!params.SKIP_TESTS) {
            sh 'mvn test'
        }
    }
}
```

### post 动作

```groovy
post {
    always    { echo '始终执行' }
    success   { echo '构建成功时执行' }
    failure   { echo '构建失败时执行' }
    unstable  { echo '测试不稳定时执行（测试有失败但不是错误）' }
    aborted   { echo '构建被中止时执行' }
    changed   { echo '构建结果与上次不同时执行' }
    fixed     { echo '从失败变为成功时执行' }
    regression{ echo '从成功变为失败时执行' }
    cleanup   { echo '在所有 post 动作之后执行（用于清理）' }
}
```

### options 选项

```groovy
options {
    // 超时控制
    timeout(time: 1, unit: 'HOURS')

    // 并发控制
    disableConcurrentBuilds(abortPrevious: true)  // 新构建触发时取消旧构建

    // 构建历史保留
    buildDiscarder(logRotator(
        numToKeepStr:         '20',   // 保留最近 20 次
        artifactNumToKeepStr: '5'     // 产物只保留最近 5 次
    ))

    // 跳过默认 Checkout（手动控制检出时机）
    skipDefaultCheckout()

    // 时间戳
    timestamps()

    // ANSI 彩色输出（需 AnsiColor 插件）
    ansiColor('xterm')

    // 重试（适用于 Stage 级别）
    retry(3)
}
```

---

## Scripted Pipeline

适合需要复杂逻辑、循环、动态 Stage 的场景。

```groovy
node('linux') {
    def mvnHome = tool 'Maven3'
    def javaHome = tool 'JDK17'

    withEnv(["PATH+MVN=${mvnHome}/bin", "JAVA_HOME=${javaHome}"]) {
        try {
            stage('Checkout') {
                checkout scm
            }

            stage('Build') {
                sh 'mvn clean package -DskipTests'
            }

            // 动态生成 Stage（Declarative 不支持）
            def services = ['auth-service', 'user-service', 'order-service']
            stage('Deploy Services') {
                for (svc in services) {
                    sh "kubectl set image deployment/${svc} ${svc}=${IMAGE}:${BUILD_NUMBER}"
                }
            }

        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            throw e
        } finally {
            cleanWs()
        }
    }
}
```

---

## 常用内置变量

| 变量 | 说明 |
|------|------|
| `env.BUILD_NUMBER` | 当前构建编号 |
| `env.BUILD_ID` | 构建 ID（同 BUILD_NUMBER） |
| `env.BUILD_URL` | 构建的完整 URL |
| `env.JOB_NAME` | Job 名称 |
| `env.JOB_BASE_NAME` | Job 名称（不含路径） |
| `env.WORKSPACE` | 工作目录路径 |
| `env.BRANCH_NAME` | 分支名（Multibranch Pipeline） |
| `env.TAG_NAME` | Tag 名（Tag 触发时） |
| `env.GIT_COMMIT` | 当前 Git commit SHA |
| `env.GIT_BRANCH` | Git 分支 |
| `env.CHANGE_ID` | PR/MR 编号 |
| `currentBuild.result` | 当前构建结果（SUCCESS/FAILURE/UNSTABLE） |
| `currentBuild.currentResult` | 实时构建结果 |
| `currentBuild.duration` | 构建耗时（毫秒） |
| `currentBuild.displayName` | 构建显示名 |

---

## 常用 Steps

### 文件与目录

```groovy
// 创建目录
sh 'mkdir -p target/reports'

// 写文件
writeFile file: 'version.txt', text: "1.0.${BUILD_NUMBER}"

// 读文件
def content = readFile 'config.properties'

// 读 JSON
def data = readJSON file: 'package.json'
echo "Version: ${data.version}"

// 读 YAML
def cfg = readYaml file: 'values.yaml'

// 文件存在检查
if (fileExists('dist/index.html')) {
    archiveArtifacts artifacts: 'dist/**'
}

// 复制文件
sh 'cp -r dist/ /var/www/html/'
```

### 归档与测试报告

```groovy
// 归档构建产物
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

// JUnit 测试报告
junit 'target/surefire-reports/**/*.xml'

// HTML 报告（需 HTML Publisher 插件）
publishHTML([
    allowMissing:         false,
    alwaysLinkToLastBuild: true,
    keepAll:              true,
    reportDir:            'target/site/jacoco',
    reportFiles:          'index.html',
    reportName:           'Code Coverage'
])

// 代码覆盖率（需 JaCoCo 插件）
jacoco execPattern: 'target/jacoco.exec'
```

### 通知

```groovy
// 邮件（需配置 SMTP）
mail(
    to:      'dev@example.com',
    subject: "Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
    body:    "详情：${env.BUILD_URL}"
)

// 企业微信（需 Qy Wechat Notification 插件）
qyWechatNotification(
    webhookUrl: 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx',
    message:    "Jenkins 构建 ${currentBuild.result}"
)

// 钉钉（需 DingTalk 插件）
dingtalk(
    robot: 'dingtalk-robot-id',
    type:  'MARKDOWN',
    title: "Jenkins 构建通知",
    text:  ["# ${env.JOB_NAME} 构建${currentBuild.result == 'SUCCESS' ? '成功' : '失败'}"]
)
```

### 用户输入

```groovy
stage('Confirm Deploy to Prod') {
    steps {
        script {
            // 等待用户确认（超时自动终止）
            def result = input(
                message: '确认部署到生产环境？',
                ok:      '确认部署',
                submitter: 'admin,ops-team',    // 只允许这些用户操作
                parameters: [
                    choice(name: 'ROLLBACK_VERSION', choices: ['1.0', '0.9', '0.8'], description: '回滚版本（取消时用）')
                ]
            )
            echo "用户选择：${result}"
        }
    }
}
```

---

## 触发器配置

### SCM 轮询

```groovy
triggers {
    // 每 5 分钟检查一次 SCM 变更
    pollSCM('H/5 * * * *')
}
```

### 定时构建（Cron 语法）

```groovy
triggers {
    // 每天凌晨 2 点
    cron('0 2 * * *')

    // 工作日每小时的第 30 分钟（H 表示哈希随机，避免同时触发）
    cron('H 30 * * 1-5')
}
```

| Cron 表达式 | 说明 |
|------------|------|
| `H * * * *` | 每小时（随机分钟） |
| `H/15 * * * *` | 每 15 分钟 |
| `H 8-17 * * 1-5` | 工作日 8-17 点每小时 |
| `@daily` | 每天 |
| `@weekly` | 每周 |

### Webhook 触发（GitLab/GitHub）

```groovy
// 需安装对应插件（Generic Webhook Trigger / GitLab / GitHub）
triggers {
    gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All')
}
```

GitLab 配置步骤：
1. Jenkins 安装 **GitLab** 插件
2. GitLab 项目 → Settings → Webhooks
3. URL 填 `http://jenkins:8080/project/your-job`
4. 勾选 Push events、Merge request events
5. 添加 Secret Token（对应 Jenkins 中的配置）

---

## 多分支 Pipeline（Multibranch Pipeline）

Multibranch Pipeline 会自动扫描代码仓库的所有分支和 PR，为每个分支单独创建 Job，对应分支根目录下的 `Jenkinsfile`。

**创建步骤：**
1. New Item → Multibranch Pipeline
2. Branch Sources → 选择 Git/GitHub/GitLab
3. 配置仓库 URL 和凭据
4. Scan Multibranch Pipeline Triggers → 设置扫描间隔

**典型 Jenkinsfile 分支策略：**

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps {
                sh './deploy.sh dev'
            }
        }

        stage('Deploy to Staging') {
            when { branch 'release/*' }
            steps {
                sh './deploy.sh staging'
            }
        }

        stage('Deploy to Prod') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                sh './deploy.sh prod'
            }
        }
    }
}
```

---

## Jenkins + Docker 集成

### 在 Pipeline 中构建 Docker 镜像

```groovy
pipeline {
    agent any

    environment {
        REGISTRY   = 'registry.cn-hangzhou.aliyuncs.com'
        IMAGE_NAME = "${REGISTRY}/my-org/my-app"
        IMAGE_TAG  = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Build Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}", "--build-arg ENV=prod .")
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'aliyun-registry-creds') {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker pull ${IMAGE_NAME}:${IMAGE_TAG}
                    docker stop my-app || true
                    docker rm   my-app || true
                    docker run -d \\
                        --name my-app \\
                        --restart unless-stopped \\
                        -p 8080:8080 \\
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
```

### 使用 Docker Agent 运行构建

```groovy
pipeline {
    agent none   // 顶层不分配 Agent

    stages {
        stage('Build Java') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args  '-v $HOME/.m2:/root/.m2'  // 挂载 Maven 缓存
                }
            }
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Build Frontend') {
            agent {
                docker {
                    image 'node:20-alpine'
                    args  '-v $HOME/.npm:/root/.npm'
                }
            }
            steps {
                sh 'npm ci && npm run build'
            }
        }
    }
}
```

---

## Jenkins + Kubernetes 集成

使用 **Kubernetes** 插件在 K8s 集群中动态创建 Pod 作为构建 Agent，构建完成后自动销毁。

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  - name: docker
    image: docker:dind
    securityContext:
      privileged: true
  volumes:
  - name: maven-cache
    persistentVolumeClaim:
      claimName: maven-cache-pvc
'''
        }
    }

    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build Image') {
            steps {
                container('docker') {
                    sh 'docker build -t my-app:latest .'
                }
            }
        }
    }
}
```

---

## 常用插件推荐

| 分类 | 插件名 | 说明 |
|------|--------|------|
| **SCM** | Git, GitHub, GitLab | 代码仓库集成 |
| **Pipeline** | Pipeline, Multibranch Pipeline | Pipeline 核心 |
| **UI** | Blue Ocean | 现代化 Pipeline 可视化界面 |
| **构建工具** | Maven Integration, NodeJS, Gradle | 构建工具集成 |
| **Docker** | Docker, Docker Pipeline | Docker 构建和发布 |
| **K8s** | Kubernetes | K8s 动态 Agent |
| **代码质量** | SonarQube Scanner | 代码静态分析 |
| **测试报告** | JUnit, HTML Publisher, JaCoCo | 测试和覆盖率报告 |
| **制品** | Nexus Artifact Uploader, Artifactory | 制品管理 |
| **通知** | Email Extension, DingTalk, Qy Wechat | 消息通知 |
| **凭据** | Credentials Binding | 安全注入凭据 |
| **安全** | Role-based Authorization Strategy | 基于角色的权限控制 |
| **时间戳** | Timestamper | 构建日志添加时间戳 |
| **工作空间** | Workspace Cleanup | 构建后清理工作目录 |
| **参数** | Extended Choice Parameter | 扩展参数类型 |

---

## 分布式构建（Master-Agent 架构）

Jenkins Master 负责调度，构建任务在 Agent（Worker）上执行。

### 添加 Agent（SSH 方式）

1. **Manage Jenkins → Nodes → New Node**
2. 填写节点名、Remote root directory（如 `/home/jenkins`）
3. Labels：用于 Pipeline 中的 `agent { label '...' }`
4. Launch method：**Launch agents via SSH**
5. 填写 Agent 机器的 IP、SSH 凭据

Agent 机器需要预先安装 Java：
```shell
sudo apt install -y openjdk-17-jre
sudo useradd -m -s /bin/bash jenkins
```

### 添加 Agent（JNLP/Inbound 方式，适合容器）

```shell
# 在 Agent 机器上执行（从 Jenkins UI 获取命令）
java -jar agent.jar \
  -url http://jenkins-master:8080/ \
  -secret <secret-token> \
  -name "docker-agent" \
  -workDir "/home/jenkins/workspace"
```

### 节点标签策略

```groovy
// 只在有 docker 标签的节点运行
agent { label 'docker' }

// 在有 linux 和 high-memory 标签的节点运行
agent { label 'linux && high-memory' }

// 在 linux 或 mac 节点上运行
agent { label 'linux || mac' }
```

---

## 共享库（Shared Library）

共享库允许跨 Job 复用 Pipeline 代码，将公共逻辑提取为库函数。

### 目录结构

```
jenkins-shared-library/
├── vars/                   # 全局变量/函数（在 Pipeline 中直接调用）
│   ├── buildMaven.groovy
│   ├── deployToK8s.groovy
│   └── notifyDingTalk.groovy
├── src/                    # 辅助类（Groovy 类）
│   └── org/example/
│       └── Utils.groovy
└── resources/              # 静态资源
    └── templates/
        └── deploy.yaml
```

### 编写共享函数（vars/buildMaven.groovy）

```groovy
def call(Map config = [:]) {
    def goals = config.get('goals', 'clean package')
    def skipTests = config.get('skipTests', false)

    sh "mvn ${goals}${skipTests ? ' -DskipTests' : ''}"
}
```

### 注册共享库

**Manage Jenkins → System → Global Pipeline Libraries**

```
Name:           shared-lib
Default version: main
Retrieval method: Modern SCM → Git
Project Repository: http://gitlab.example.com/devops/shared-lib.git
Credentials: gitlab-creds
```

### 在 Pipeline 中使用

```groovy
@Library('shared-lib') _          // 引入共享库

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                buildMaven goals: 'clean package', skipTests: true
            }
        }
        stage('Deploy') {
            steps {
                deployToK8s namespace: 'production', image: "myapp:${BUILD_NUMBER}"
            }
        }
    }
    post {
        failure {
            notifyDingTalk message: "构建失败：${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

---

## 安全配置

### 基于角色的权限控制（Role Strategy 插件）

1. 安装 **Role-based Authorization Strategy** 插件
2. **Manage Jenkins → Security → Authorization** → 选择 **Role-Based Strategy**
3. **Manage Jenkins → Manage and Assign Roles**

| 角色 | 权限 |
|------|------|
| admin | 所有权限 |
| developer | 构建、查看、取消 |
| viewer | 只读查看 |
| deployer | 仅允许触发特定 Job |

### CSRF 保护

Jenkins 默认启用 CSRF 保护，API 调用需携带 Crumb Token：

```shell
# 获取 Crumb
CRUMB=$(curl -s -u admin:token \
  "http://jenkins:8080/crumbIssuer/api/json" \
  | jq -r '.crumb')

# 触发构建
curl -s -X POST \
  -u admin:token \
  -H "Jenkins-Crumb: ${CRUMB}" \
  "http://jenkins:8080/job/my-job/build"
```

---

## Jenkins API 使用

### REST API 常用操作

```shell
# 获取 Job 列表
curl -s -u admin:token http://jenkins:8080/api/json | jq '.jobs[].name'

# 触发构建（无参数）
curl -X POST -u admin:token \
  http://jenkins:8080/job/my-job/build

# 触发参数化构建
curl -X POST -u admin:token \
  "http://jenkins:8080/job/my-job/buildWithParameters?BRANCH=main&ENV=prod"

# 查询构建状态
curl -s -u admin:token \
  http://jenkins:8080/job/my-job/lastBuild/api/json | jq '.result'

# 获取构建日志
curl -s -u admin:token \
  http://jenkins:8080/job/my-job/lastBuild/consoleText

# 停止构建
curl -X POST -u admin:token \
  http://jenkins:8080/job/my-job/lastBuild/stop
```

### 使用 API Token

1. 用户头像 → Configure → API Token → Add new Token
2. 保存 Token（只显示一次）
3. 用于 Basic Auth：`curl -u username:token`

---

## 常见完整示例

### Java Maven 项目 CI/CD

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk   'JDK17'
    }

    environment {
        NEXUS_URL     = 'http://nexus.example.com'
        SONAR_URL     = 'http://sonar.example.com'
        DOCKER_REGISTRY = 'registry.example.com'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.APP_VERSION = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    echo "Version: ${env.APP_VERSION}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit       'target/surefire-reports/**/*.xml'
                    jacoco      execPattern: 'target/jacoco.exec',
                                classPattern: 'target/classes',
                                sourcePattern: 'src/main/java'
                }
            }
        }

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build & Push Docker Image') {
            when { anyOf { branch 'main'; branch 'develop' } }
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/my-app:${APP_VERSION}-${BUILD_NUMBER}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'registry-creds') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps {
                sh "kubectl set image deployment/my-app my-app=${DOCKER_REGISTRY}/my-app:${APP_VERSION}-${BUILD_NUMBER} -n dev"
                sh "kubectl rollout status deployment/my-app -n dev --timeout=120s"
            }
        }

        stage('Deploy to Prod') {
            when { branch 'main' }
            steps {
                input message: "确认部署 v${APP_VERSION} 到生产环境？", ok: '确认'
                sh "kubectl set image deployment/my-app my-app=${DOCKER_REGISTRY}/my-app:${APP_VERSION}-${BUILD_NUMBER} -n prod"
                sh "kubectl rollout status deployment/my-app -n prod --timeout=300s"
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', message: "✅ 构建成功：${JOB_NAME} #${BUILD_NUMBER} - ${APP_VERSION}")
        }
        failure {
            slackSend(color: 'danger', message: "❌ 构建失败：${JOB_NAME} #${BUILD_NUMBER}")
        }
        always {
            cleanWs()
        }
    }
}
```

### Node.js 前端项目 CI/CD

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args  '-v $HOME/.npm:/root/.npm'
        }
    }

    environment {
        CI = 'true'
    }

    stages {
        stage('Install') {
            steps {
                sh 'npm ci --prefer-offline'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Test') {
            steps {
                sh 'npm run test -- --coverage --watchAll=false'
            }
            post {
                always {
                    publishHTML([
                        reportDir:   'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName:  'Coverage Report'
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh 'aws s3 sync dist/ s3://my-bucket/ --delete'
                sh 'aws cloudfront create-invalidation --distribution-id XXXXX --paths "/*"'
            }
        }
    }
}
```

---

## 备份与维护

### 备份 Jenkins 数据

```shell
# 备份 JENKINS_HOME（排除构建日志和临时文件）
tar -czf jenkins-backup-$(date +%Y%m%d).tar.gz \
  --exclude='jobs/*/builds' \
  --exclude='workspace' \
  --exclude='war' \
  --exclude='.gradle' \
  /var/jenkins_home

# 使用 ThinBackup 插件（推荐）
# Manage Jenkins → ThinBackup → 配置备份路径和计划
```

关键配置文件：
```
$JENKINS_HOME/
├── config.xml                    # 主配置
├── credentials.xml               # 凭据
├── jobs/                         # Job 配置
│   └── my-job/
│       └── config.xml
├── plugins/                      # 插件
└── secrets/                      # 加密密钥（务必备份）
```

### 升级 Jenkins

```shell
# Docker 方式升级
docker pull jenkins/jenkins:lts-jdk17
docker stop jenkins
docker rm jenkins
# 重新运行（数据卷不变，数据保留）
docker run -d --name jenkins ... jenkins/jenkins:lts-jdk17
```

### 清理构建历史

```groovy
// 在 Job 的 options 中配置
options {
    buildDiscarder(logRotator(
        daysToKeepStr:        '30',   // 保留 30 天
        numToKeepStr:         '100',  // 最多保留 100 次
        artifactDaysToKeepStr:'7',    // 产物保留 7 天
        artifactNumToKeepStr: '5'     // 产物最多 5 次
    ))
}
```

---

## 常见问题排查

### 构建卡住不动

```shell
# 查看 Jenkins 线程 dump
curl -u admin:token http://jenkins:8080/threadDump

# 检查 Executor 是否被占用
# Manage Jenkins → Nodes → 查看各节点 Executor 状态
```

### OutOfMemoryError

```shell
# 增大 JVM 堆内存
# Docker 方式：
docker run -e JAVA_OPTS="-Xmx2g -Xms512m" jenkins/jenkins:lts-jdk17

# systemd 方式：
# /etc/default/jenkins
JAVA_ARGS="-Xmx2g -Xms512m -XX:+UseG1GC"
```

### 插件更新失败

```shell
# 手动下载插件（.hpi 文件）从 https://plugins.jenkins.io
# 上传：Manage Jenkins → Plugins → Advanced → Deploy Plugin
```

### 工作空间权限问题

```shell
# 修复 Jenkins 工作目录权限
chown -R jenkins:jenkins /var/lib/jenkins/workspace/
chmod -R 755 /var/lib/jenkins/workspace/
```

### Pipeline 语法验证

Jenkins 内置语法校验工具：
- `http://jenkins:8080/pipeline-syntax/` — Pipeline Snippet Generator（代码片段生成器）
- `http://jenkins:8080/directive-generator/` — Declarative Directive Generator
- 在 Jenkinsfile 编辑器中使用 **Validate** 按钮

---

## 最佳实践

1. **Pipeline as Code**：所有 Pipeline 定义写在 `Jenkinsfile` 中并提交到代码仓库，而不是在 UI 中配置。
2. **最小权限原则**：为 Agent 和服务账号配置最小必要权限，避免使用 root。
3. **凭据管理**：所有密码、Token、密钥通过 Jenkins Credentials 管理，严禁硬编码到 Jenkinsfile。
4. **构建隔离**：每次构建开始前清理工作空间（`cleanWs()`），避免上次构建残留干扰。
5. **超时设置**：为每个 Pipeline 和 Stage 设置合理的 `timeout`，防止僵尸构建占用资源。
6. **并发控制**：使用 `disableConcurrentBuilds()` 防止同一 Job 并发运行导致资源竞争。
7. **制品版本化**：镜像和构建产物使用 `版本号-构建号` 作为 Tag，保证可追溯。
8. **共享库**：将公共构建逻辑提取到 Shared Library，避免在多个 Jenkinsfile 中重复代码。
9. **蓝绿/金丝雀发布**：生产部署配合 `input` 步骤做人工确认，重要版本分阶段发布。
10. **监控告警**：接入 Prometheus + Grafana 监控 Jenkins 指标（构建队列、构建时长、失败率）。

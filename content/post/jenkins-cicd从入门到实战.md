---
title: "Jenkins CI/CD 从入门到实战：自动化部署不再难"
description: "Jenkins 是最流行的持续集成工具之一。从安装到配置 Pipeline，手把手带你搭建自动化部署流程。"
date: 2026-03-25
tags:
  - Jenkins
  - CI/CD
  - 运维
  - Docker
---
> Jenkins 虽然老了，但仍然是企业里用得最多的 CI/CD 工具。掌握 Jenkins Pipeline 是全栈/测试工程师的加分项。

* * *

## 一、安装 Jenkins 

### Docker 一键安装（推荐） 
    
    
    docker run -d \
      --name jenkins \
      -p 8080:8080 \
      -p 50000:50000 \
      -v jenkins_home:/var/jenkins_home \
      -v /var/run/docker.sock:/var/run/docker.sock \
      --restart unless-stopped \
      jenkins/jenkins:lts
    

> 挂载 docker.sock 是为了在 Pipeline 里能用 Docker。

### 初始化 

  1. 访问 `http://your-ip:8080`
  2. 查看初始密码：`docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`
  3. 安装推荐插件
  4. 创建管理员账户



### 推荐安装的插件 

插件 | 用途  
---|---  
Git | 代码拉取  
Pipeline | 流水线定义  
Docker Pipeline | 在 Pipeline 中使用 Docker  
Credentials | 凭证管理  
Blue Ocean | 可视化 Pipeline（已弃用但好看）  
Chinese Localization | 中文界面  
  
* * *

## 二、基础概念 

### Jenkins 核心术语 

术语 | 说明  
---|---  
**Job** | 一个构建任务  
**Pipeline** | 多阶段流水线（推荐方式）  
**Stage** | 流水线的一个阶段  
**Step** | 阶段中的一个具体操作  
**Node / Agent** | 执行构建的机器  
**Workspace** | 构建的工作目录  
  
### Pipeline vs 传统 Job 

**传统 Freestyle Job：** 在网页上点按钮配置，适合简单任务，但不方便版本管理。

**Pipeline：** 用代码（Jenkinsfile）定义流程，提交到代码仓库，**推荐使用。**

* * *

## 三、Pipeline 实战 

### Jenkinsfile 示例：Python 项目 
    
    
    pipeline {
        agent any
        
        environment {
            PYTHON_VERSION = '3.11'
        }
        
        stages {
            stage('拉取代码') {
                steps {
                    git 'https://github.com/yourname/yourproject.git'
                }
            }
            
            stage('安装依赖') {
                steps {
                    sh 'pip install -r requirements.txt'
                }
            }
            
            stage('代码检查') {
                steps {
                    sh 'flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics'
                }
            }
            
            stage('运行测试') {
                steps {
                    sh 'pytest tests/ -v --junitxml=report.xml'
                }
                post {
                    always {
                        junit 'report.xml'
                    }
                }
            }
            
            stage('构建镜像') {
                steps {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                }
            }
            
            stage('部署') {
                steps {
                    sh 'docker stop myapp || true'
                    sh 'docker rm myapp || true'
                    sh 'docker run -d --name myapp -p 8000:8000 myapp:${BUILD_NUMBER}'
                }
            }
        }
        
        post {
            success {
                echo '✅ 部署成功！'
            }
            failure {
                echo '❌ 部署失败，请检查日志。'
            }
            always {
                cleanWs()
            }
        }
    }
    

### Jenkinsfile 示例：前端项目 
    
    
    pipeline {
        agent any
        
        stages {
            stage('安装依赖') {
                steps {
                    sh 'npm install'
                }
            }
            
            stage('代码检查') {
                steps {
                    sh 'npm run lint'
                }
            }
            
            stage('构建') {
                steps {
                    sh 'npm run build'
                }
            }
            
            stage('部署到 Nginx') {
                steps {
                    sh 'docker cp dist/ nginx-container:/usr/share/nginx/html/'
                }
            }
        }
    }
    

* * *

## 四、常用操作 

### 凭证管理 
    
    
    // 使用 Git 凭证
    checkout([
        $class: 'GitSCM',
        branches: [[name: 'main']],
        userRemoteConfigs: [[
            url: 'https://github.com/yourname/project.git',
            credentialsId: 'git-credentials'
        ]]
    ])
    
    // 使用 SSH 密钥
    withCredentials([sshUserPrivateKey(
        credentialsId: 'ssh-key',
        keyFileVariable: 'SSH_KEY'
    )]) {
        sh "ssh -i $SSH_KEY user@server 'deploy.sh'"
    }
    

### 参数化构建 
    
    
    pipeline {
        agent any
        parameters {
            string(name: 'BRANCH', defaultValue: 'main', description: '构建分支')
            choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: '部署环境')
            booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: '跳过测试')
        }
        
        stages {
            stage('拉取代码') {
                steps {
                    git branch: params.BRANCH, url: 'https://github.com/yourname/project.git'
                }
            }
            
            stage('测试') {
                when {
                    expression { !params.SKIP_TESTS }
                }
                steps {
                    sh 'pytest tests/'
                }
            }
            
            stage('部署') {
                steps {
                    echo "部署到 ${params.ENV} 环境"
                }
            }
        }
    }
    

### 定时构建 
    
    
    // Poll SCM：每 5 分钟检查一次代码变更
    triggers {
        pollSCM('H/5 * * * *')
    }
    
    // 定时构建：每天上午 10 点
    triggers {
        cron('H 10 * * *')
    }
    

* * *

## 五、多节点（Agent）配置 

如果你有多台机器：

### 添加 Agent 

  1. **Manage Jenkins** → **Nodes** → **New Node**
  2. 设置节点名称、工作目录
  3. 选择连接方式（SSH）
  4. 配置 SSH 凭证



### 在 Pipeline 中指定节点 
    
    
    pipeline {
        agent none
        
        stages {
            stage('构建') {
                agent { label 'build-server' }
                steps {
                    sh 'make build'
                }
            }
            
            stage('测试') {
                agent { label 'test-server' }
                steps {
                    sh 'make test'
                }
            }
            
            stage('部署') {
                agent { label 'prod-server' }
                steps {
                    sh 'make deploy'
                }
            }
        }
    }
    

* * *

## 六、Webhook 自动触发 

### GitHub / GitLab 集成 

  1. Jenkins 安装插件：`Generic Webhook Trigger` 或 `GitLab Plugin`
  2. 在 Jenkins Job 中配置 Webhook URL
  3. 在 GitHub/GitLab 仓库设置中添加 Webhook


    
    
    triggers {
        GenericTrigger(
            genericVariables: [[key: 'ref', value: '$.ref']],
            causeString: 'Triggered by push',
            printContributedVariables: true,
            printPostContent: true
        )
    }
    

* * *

## 七、实用技巧 

### 并行构建 
    
    
    stage('并行测试') {
        parallel {
            stage('单元测试') {
                steps { sh 'pytest tests/unit/' }
            }
            stage('集成测试') {
                steps { sh 'pytest tests/integration/' }
            }
        }
    }
    

### 构建超时 
    
    
    stage('测试') {
        options {
            timeout(time: 30, unit: 'MINUTES')
        }
        steps {
            sh 'pytest tests/'
        }
    }
    

### 构建通知 
    
    
    post {
        success {
            // 飞书/钉钉/企业微信通知
            sh 'curl -X POST "$WEBHOOK_URL" -H "Content-Type: application/json" -d \'{"msg_type":"text","content":{"text":"✅ 构建成功: $JOB_NAME #${BUILD_NUMBER}"}}\''
        }
        failure {
            sh 'curl -X POST "$WEBHOOK_URL" -H "Content-Type: application/json" -d \'{"msg_type":"text","content":{"text":"❌ 构建失败: $JOB_NAME #${BUILD_NUMBER}"}}\''
        }
    }
    

* * *

## 八、Jenkins 维护 

### 清理旧构建 
    
    
    // Pipeline 中配置
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    

### 备份与恢复 
    
    
    # 备份（Docker 方式）
    docker run --rm -v jenkins_home:/var/jenkins_home -v $(pwd):/backup \
      busybox tar czf /backup/jenkins-backup.tar.gz /var/jenkins_home
    
    # 恢复
    docker run --rm -v jenkins_home:/var/jenkins_home -v $(pwd):/backup \
      busybox tar xzf /backup/jenkins-backup.tar.gz -C /
    

* * *

_Jenkins 虽然界面不够现代，但功能强大、插件丰富、社区成熟。学会写 Pipeline，你就掌握了 CI/CD 的核心能力。_
# jenkins_demo
jenkins demo project

learnt from bilibili:https://www.bilibili.com/video/BV1Z2qYBSERP/?spm_id_from=333.337.search-card.all.click&vd_source=d454820f37020de1127785590f490387

```jenkins
docker run --detach \
    --restart always \
    --network jenkins-demo \
    --name jenkins \
    --publish 8080:8080 \
    --publish 50000:50000 \
    --volume ~/docker/jenkins/jenkins_home:/var/jenkins_home \
    jenkins/jenkins


docker run -itd \
    --restart always \
    --network jenkins-demo \
    --name alpine \
    --publish 127.0.0.1:2376:2375 \
    --volume /var/run/docker.sock:/var/run/docker.sock \
    alpine/socat tcp-listen:2375,fork,reuseaddr unix-connect:/var/run/docker.sock

docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword


pipeline {
    // 1. 改回通用代理，不再依赖外部 Docker 插件
    agent any

    // 2. 引入你刚才在 Tools 里配置好的 JDK 和 Maven 名字
    tools {
        jdk 'jdk21'
        maven 'maven3.9'
    }

    environment {
        // 引用你之前在凭据管理里创建的密码/私钥 ID
        SSH_CREDENTIAL_ID = 'github-ssh' 
    }

    stages {
        stage('Checkout') {
            steps {
                echo '=== 正在安全地从 GitHub 拉取代码 ==='
                checkout scmGit(
                    branches: [[name: '*/main']], 
                    extensions: [], 
                    userRemoteConfigs: [[
                        credentialsId: "${SSH_CREDENTIAL_ID}", 
                        url: 'git@github.com:KellyChen1/jenkins_demo.git'
                    ]]
                )
            }
        }

        stage('Verify Environment') {
            steps {
                echo '=== 检查当前构建环境的软件版本 ==='
                // 此时 Jenkins 会自动下载并临时配置好这两款工具的环境变量
                sh 'java -version'
                sh 'mvn -v'
            }
        }

        stage('Compile & Test') {
            steps {
                echo '=== 进入 demo 目录并执行单元测试与代码编译 ==='
                // 🛠️ 关键修复：指定在 demo 子目录下执行 Maven 命令
                dir('demo') {
                    sh 'mvn clean test'
                }
            }
        }

        stage('Package') {
            steps {
                echo '=== 进入 demo 目录并将 Java 项目打包为 Jar ==='
                // 🛠️ 关键修复：同样在 demo 子目录下执行打包
                dir('demo') {
                    sh 'mvn package -Dmaven.test.skip=true'
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                echo '=== 归档并保存构建产物 ==='
                // 🛠️ 关键修复：因为打包在 demo/target 下，所以路径也要加上 demo/ 路径
                archiveArtifacts artifacts: 'demo/target/*.jar', followSymlinks: false
            }
        }
    }

    post {
        success {
            echo '🎉 恭喜！整个 Java 项目流水线构建成功！'
        }
        failure {
            echo '❌ 糟糕，构建失败了，请查看上方日志排查错误。'
        }
    }
}



 
```

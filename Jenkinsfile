pipeline {
    agent any  // 统一使用4空格缩进
    
    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        BUILD_DIR = "${WORKSPACE}/target"
        DOCKER_IP = "192.168.181.138"
        DOCKER_DIR = "/afs/ry_compose/ry"
        PACKAGE_JAR = 'ruoyi.jar'
        SSH_CREDENTIALS_ID = '3df4cfee-0317-477b-b534-baea92760a56'
        SSH_USER = "root"
    }

    tools { 
        maven 'maven3.9.11' 
        jdk 'java11' 
    }

    stages {
        stage('拉取代码') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    echo '拉取代码中'
                    checkout scm
                }
            }
            post {
                always {
                    echo "拉取代码结束"
                }
            }
        }

        stage('打包') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'mvn -v'
                    sh 'mvn clean package'
                }

                script {
                    def currentDir = pwd()
                    echo "Jenkins当前工作目录: ${currentDir}"
                    sh "ls -l ${currentDir}/target"
                    def File = findFiles(glob: "**/*.jar")[0]
                    echo "${File.getName()}"
                    if (File) {
                        env.PACKAGE_JAR = File.getName()
                        echo "找到 JAR 包: ${env.PACKAGE_JAR}"
                    } else {
                        error "打包后未找到 JAR 文件！请检查 Maven 构建日志。"
                    }
                } 
            }
            post {
                always {
                    echo "打包结束"
                }
            }
        }
    
        stage('运行docker compose') {
            steps {
                script {
                    if (!env.PACKAGE_JAR) {
                        error "JAR 包名未知，无法部署！请确保前一阶段成功。"
                    }
                }
                echo "部署 ${env.PACKAGE_JAR} 到 ${env.DOCKER_IP}..."
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                        rsync -rpz ${env.BUILD_DIR}/${env.PACKAGE_JAR} ${env.SSH_USER}@${env.DOCKER_IP}:${env.DOCKER_DIR}
                        ssh ${env.SSH_USER}@${env.DOCKER_IP} "
                            cd /afs/ry_compose/ &&
                            docker compose down --remove-orphans &&
                            docker compose up -d --build --force-recreate
                        "
                    """
                }
            }
            post {
                success {
                    echo "运行阶段成功完成。"
                }
                failure {
                    echo "运行失败，请检查日志！"
                }
                aborted {
                    echo "运行阶段被中止。"
                }
            }  
        }
    }  // stages 结束

    post {  // 流水线级别的 post
        success {
            echo '所有阶段成功！应用程序已部署并运行。'
        }
        failure {
            echo '流水线执行失败。请检查日志以获取详细信息。'
        }
        unstable {
            echo '流水线执行不稳定，可能存在警告。'
        }
        aborted {
            echo '流水线被中止。'
        }
        always {
            cleanWs()
        }
    }
}

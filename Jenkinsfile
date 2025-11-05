pipeline {
  agent any
	
	options { // 可选：定义流水线范围内的选项
  		// 例如：
  		timestamps() // 在控制台输出中显示时间戳
  		disableConcurrentBuilds() // 禁止并发执行，同一时间只能有一个构建运行
  		skipDefaultCheckout(true) // 跳过默认的 SCM checkout，如果你想在某个阶段手动 checkout
	}

  environment {
      BUILD_DIR = "${WORKSPACE}/target"

      // docker 的部署信息
      DOCKER_IP = "192.168.181.138"
      DOCKER_DIR = "/afs/ry_compose/ry"

      // 定义一个 PACKAGE_WAR 变量，用于在运行时动态获取，初始留空
      PACKAGE_JAR = 'ruoyi.jar'

      // SSH 凭据的 Jenkins ID (需要在 Jenkins Credential Manager 中配置)
      // /manage/credentials/store/system/domain/_/newCredentials
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
          // finally 逻辑可以在 post 块中处理
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
              ssh ${env.SSH_USER}@${env.DOCKER_IP} "cd /afs/ry_compose && docker compose up -d"
            """
          }
        }

        post {
          always {
              echo "运行阶段结束。"
          }
          failure {
              echo "运行失败，请检查日志！"
          }
        }  
      }

    post {
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
            // 清理工作空间，避免下次构建遇到旧文件 (可选，但推荐)
            cleanWs()
        }
    }
  }
}

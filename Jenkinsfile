pipeline {
   agent any

   stages {
      stage('Verify Branch') {
         steps {
            echo "$GIT_BRANCH"
         }
      }
      stage('Docker Build') { 
         steps {
            powershell(script: """
               cd azure-vote/
               docker images -a
               docker build -t jenkins-pipeline .
               docker images -a
               cd ..
            """)
         }
      }
      stage('Start test app') {
         steps {
            powershell(script: """
               docker-compose up -d
               ./scripts/test_container.sh
            """)
         }
         post {
            success {
               echo "App started successfully :)"
            }
            failure {
               // echo "App failed to start :(",
               notify("FAIL")
            }
         }
      }
      stage('Run Tests') {
         steps {
            powershell(script: """
               pytest ./tests/test_sample.py
            """)
         }
      }
      stage('Stop test app') {
         steps {
            powershell(script: """
               docker-compose down
            """)
         }
      }
      stage('Container Scanning') {
         parallel {
            stage('Run Anchore') {
               steps {
                  powershell(script: """
                     Write-Output "blackdentech/jenkins-course" > anchore_images
                  """)
                  anchore bailOnFail: false, bailOnPluginFail: false, name: 'anchore_images'
               }
            }
            stage('Run Trivy') {
               steps {
                  sleep(time: 30, unit: 'SECONDS')
                  // powershell(script: """
                  // C:\\Windows\\System32\\wsl.exe -- sudo trivy blackdentech/jenkins-course
                  // """)
               }
            }
         }
      }
      stage('Deploy to QA') {
         environment {
            ENVIRONMENT = 'qa'
         }
         steps {
            echo "Deploying to ${ENVIRONMENT}"
            acsDeploy(
               azureCredentialsId: "jenkins_demo",
               configFilePaths: "**/*.yaml",
               containerService: "${ENVIRONMENT}-demo-cluster | AKS",
               resourceGroupName: "${ENVIRONMENT}-demo",
               sshCredentialsId: ""
            )
         }
      }
      stage('Approve PROD Deploy') {
         when {
            branch 'master'
         }
         options {
            timeout(time: 1, unit: 'HOURS') 
         }
         steps {
            input message: "Deploy?"
         }
         post {
            success {
               echo "Production Deploy Approved"
            }
            aborted {
               echo "Production Deploy Denied"
            }
         }
      }
      stage('Deploy to PROD') {
         when {
            branch 'master'
         }
         environment {
            ENVIRONMENT = 'prod'
         }
         steps {
            echo "Deploying to ${ENVIRONMENT}"
            acsDeploy(
               azureCredentialsId: "jenkins_demo",
               configFilePaths: "**/*.yaml",
               containerService: "${ENVIRONMENT}-demo-cluster | AKS",
               resourceGroupName: "${ENVIRONMENT}-demo",
               sshCredentialsId: ""
            )
         }
      }
   }
}





// notify function
def notify(status) {
   emailext body: """${status}: Job ${JOB_NAME} Build #${BUILD_NUMBER} 
   Check it out at: ${BUILD_URL} """, 
    subject: "status: Job ${JOB_NAME} Build #${BUILD_NUMBER}", 
    to: "localhost:8025", "david.debray@protonmail.com"
}

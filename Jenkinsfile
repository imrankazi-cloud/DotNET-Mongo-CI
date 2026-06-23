pipeline {
    agent any
    
     environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git CheckOut') {
            steps {
                git branch: 'main', credentialsId: 'git-token', url: 'https://github.com/imrankazi-cloud/DotNET-Mongo-CI.git'
            }
        }
        
        stage('Gitleak Scan') {
            steps {
                sh 'gitleaks detect --report-format=json --report-path=gitleaks-report.json --exit-code=1'
            }
        }
        
        stage('DotNet Build') {
            steps {
                sh 'dotnet build'
            }
        }
        
        
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        
        stage('Unit Test') {
            steps {
                echo 'dotnet test'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NoteApp \
                            -Dsonar.projectKey=NoteApp '''
                }
                
            }
        }
        
        stage('QualityGate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                   waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                       sh 'docker build -t imranawsdevops/noteapp:$IMAGE_TAG .'
                    }
                }
            }
        }
        
         stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html imranawsdevops/noteapp:$IMAGE_TAG'
            }
        }
        
        stage('Docker Push') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                       sh 'docker push imranawsdevops/noteapp:$IMAGE_TAG'
                    }
                }
            }
        }
        
        
        stage('Update Manifest File CD Repo') {
            steps {
                script {
                    cleanWs()
                    withCredentials([usernamePassword(credentialsId: 'git-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                         sh '''
                            # Clone the CD Repo
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/imrankazi-cloud/DotNET-Mongo-CD.git
                            
                            # Update the tag in manifest
                            cd DotNET-Mongo-CD
                            sed -i "s|imranawsdevops/noteapp:.*|imranawsdevops/noteapp:${IMAGE_TAG}|" Manifest/manifest.yaml
                            
                            # Confirm Changes
                            echo "Updated manifest file contents:"
                            cat Manifest/manifest.yaml
                            
                            # Commit and push the changes
                            git config user.name "Jenkins"
                            git config user.email "jenkins@example.com"
                            git add Manifest/manifest.yaml
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push origin main
                        '''
            }
            }
        }
    }
}
post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'imranarkaws@gmail.com',
                from: 'jaiswaladi246@gmail.com',
                replyTo: 'jenkins@devopsshack.com',
                mimeType: 'text/html',
               
            )
        }
    }
}
}

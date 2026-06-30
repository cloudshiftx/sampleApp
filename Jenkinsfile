def buildTag = ''

pipeline {
    agent { label 'build-agent-01' }


    stages {
        stage('Generate Tag') {
            steps {
                script {
                    def date = new Date().format('yyyyMMdd')
                    buildTag = "${date}.${env.BUILD_NUMBER}"
                    currentBuild.displayName = buildTag
                    sh "echo BUILD_TAG=${buildTag} > build.env"
                }
            }
        }

        stage('Use Tag') {
            steps {
                script {
                    echo "The build tag is: ${buildTag}"
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/cloudshiftx/sampleApp.git', branch: 'master'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarServer') {
                    sh '''
                        export PATH=$PATH:/home/danish/.dotnet/tools

                        dotnet sonarscanner begin /k:"sampleapp"

                        dotnet build -c Release

                        dotnet sonarscanner end
                    '''
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

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t sampleapp:${buildTag} ."
                }
            }
        }

        //  Trivy with HTML report
        stage('Trivy Scan') {
            steps {
                echo 'Preparing Trivy template...'
                sh '''
                    mkdir -p contrib
                    curl -s -o contrib/html.tpl https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl
                '''

                echo 'Running Trivy scan...'
                sh """
                    trivy image --scanners vuln \
                    --format template \
                    --template "@contrib/html.tpl" \
                    -o trivy-report.html \
                    sampleapp:${buildTag}
                """
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.html', fingerprint: true

                    publishHTML([
                        reportDir: '.',
                        reportFiles: 'trivy-report.html',
                        reportName: 'Trivy Security Report'
                    ])
                }
            }
        }

        stage('Push to Docker Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-login-itc', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh """
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                            docker tag sampleapp:${buildTag} ${DOCKER_USER}/sampleapp:${buildTag}
                            docker push ${DOCKER_USER}/sampleapp:${buildTag}
                        """
                    }
                }
            }
        }
    }
}

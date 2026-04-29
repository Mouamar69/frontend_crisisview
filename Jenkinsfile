pipeline {
    agent { label 'agent_node' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh 'npm run lint'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.host.url=http://sonarqube:9000"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false
                    }
                }
            }
        }

        stage('Build & Package') {
            steps {
                sh 'docker build -t crisisview-front:$BUILD_NUMBER -t crisisview-front:latest .'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag crisisview-front:latest $DOCKER_USER/crisisview-front:latest
                        docker tag crisisview-front:$BUILD_NUMBER $DOCKER_USER/crisisview-front:$BUILD_NUMBER
                        docker push $DOCKER_USER/crisisview-front:latest
                        docker push $DOCKER_USER/crisisview-front:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'DOCKER_USER=$DOCKER_USER docker compose up -d --pull always front'
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline en echec. Rollback : docker compose up -d --no-build front'
        }
        always {
            archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true
        }
    }
}

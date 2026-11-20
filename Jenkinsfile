pipeline {
    agent any
    environment {
        SERVICE = 'gateway'            // Docker Hub repository: <user>/gateway
        TRIVY_VERSION = '0.76.2'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        // bottleneck: vulnerability scanning, the build stops on a HIGH or CRITICAL with a fix available
        stage('Scan Image') {
            steps {
                sh 'docker build -t $SERVICE:scan-$BUILD_ID .'
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:$TRIVY_VERSION image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 $SERVICE:scan-$BUILD_ID'
            }
            post {
                always {
                    sh 'docker rmi $SERVICE:scan-$BUILD_ID || true'
                }
            }
        }
        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'TOKEN')]) {
                    sh 'echo $TOKEN | docker login -u $USERNAME --password-stdin'
                    sh 'docker buildx create --use --platform=linux/arm64,linux/amd64 --node multi-platform-builder-$SERVICE --name multi-platform-builder-$SERVICE'
                    sh 'docker buildx build --platform=linux/arm64,linux/amd64 --push --tag $USERNAME/$SERVICE:latest --tag $USERNAME/$SERVICE:$BUILD_ID -f Dockerfile .'
                    sh 'docker buildx rm --force multi-platform-builder-$SERVICE'
                }
            }
        }
    }
}

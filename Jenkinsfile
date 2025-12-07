pipeline {
    agent { label 'worker' }

    environment {
        DOCKERHUB = credentials('dockerhub')
        IMAGE_NAME = "alexdevops05/step2"
    }

    stages {

        stage('checkout code') {
            steps {
                echo "cloning github repository"
                git branch: 'main', url: 'https://github.com/allex-git/step2.git'
            }
        }

        stage('build image') {
            steps {
                echo "build image"
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('run tests') {
            steps {
                echo "run tests"
                sh "docker run --rm ${IMAGE_NAME}:latest npm test"
            }
        }

        stage('push to dockerHub') {
            when {
                expression { currentBuild.currentResult == "SUCCESS" }
            }
            steps {
                echo "push image to docker hub"
                sh '''
                    echo ${DOCKERHUB_PSW} | docker login -u ${DOCKERHUB_USR} --password-stdin
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }
    }

    post {
        success { echo "build and tests successfully" }
        failure { echo "tests failed" }
    }
}

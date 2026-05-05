pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/product-catalog"
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh '''
                    cd src/product-catalog
                    go build ./...
                    go test ./...
                '''
            }
        }

        stage('Code Quality') {
            steps {
                sh 'golangci-lint run src/product-catalog/...'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        docker build -t $DOCKER_IMAGE:$IMAGE_TAG src/product-catalog/
                        echo $DOCKER_TOKEN | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_IMAGE:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
                    sh '''
                        sed -i "s|image: .*product-catalog.*|image: $DOCKER_IMAGE:$IMAGE_TAG|" kubernetes/productcatalog/deploy.yaml
                        git config user.email "jenkins@ci.com"
                        git config user.name "Jenkins CI"
                        git add kubernetes/productcatalog/deploy.yaml
                        git commit -m "CI: update product-catalog image to build-$IMAGE_TAG"
                        git push https://$GIT_TOKEN@github.com/your-username/your-repo.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded. FluxCD will sync the updated manifest to EKS."
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}

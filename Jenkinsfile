pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 40, unit: 'MINUTES')
    }

    environment {
        DOCKER_USER   = 'hariprasad123456'
        AWS_REGION    = 'ap-southeast-2'
        EKS_CLUSTER   = 'my-eks-cluster'
        K8S_NAMESPACE = 'roboshop'
        SERVICES      = 'catalogue user cart shipping payment frontend'
        DB_SERVICES   = 'mongodb redis mysql rabbitmq'
    }

    stages {

        stage('Checkout') {
            steps {
                cleanWs()
                git branch: 'main', url: 'https://github.com/HarshaCloud407/K8S-Roboshop-CICD.git'
                script {
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${sha}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Show Repo Layout') {
            steps {
                sh '''#!/bin/bash
                    echo "Services: [$SERVICES]"
                    [ -n "$SERVICES" ] || { echo "ERROR: SERVICES is empty"; exit 1; }
                    echo "---- files in repo ----"
                    find . -path ./.git -prune -o -type f -print | sort
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''#!/bin/bash
                    set -e
                    for s in $SERVICES; do
                        df=$(find . -path ./.git -prune -o -type f -path "*/$s/Dockerfile" -print | head -1)
                        [ -n "$df" ] || { echo "ERROR: no Dockerfile found for $s"; exit 1; }
                        echo "Building $s from $(dirname "$df")"
                        docker build -t $DOCKER_USER/$s:$IMAGE_TAG "$(dirname "$df")"
                    done
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh '''#!/bin/bash
                        set -e
                        for s in $SERVICES; do
                            docker push $DOCKER_USER/$s:$IMAGE_TAG
                        done
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-cred',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh '''#!/bin/bash
                        set -e
                        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER \
                            --kubeconfig kubeconfig-$BUILD_NUMBER
                        export KUBECONFIG=$PWD/kubeconfig-$BUILD_NUMBER

                        find_manifest() {
                            find . -path ./.git -prune -o -type f -path "*/$1/manifest.yaml" -print | head -1
                        }

                        # namespace: use the yaml if it exists, otherwise create it
                        ns=$(find . -path ./.git -prune -o -type f -name '*namespace*.yaml' -print | head -1)
                        if [ -n "$ns" ]; then
                            kubectl apply -f "$ns"
                        else
                            kubectl get ns $K8S_NAMESPACE >/dev/null 2>&1 || kubectl create ns $K8S_NAMESPACE
                        fi

                        # databases / queue: apply as-is
                        for s in $DB_SERVICES; do
                            m=$(find_manifest $s)
                            [ -n "$m" ] || { echo "ERROR: no manifest.yaml for $s"; exit 1; }
                            kubectl apply -f "$m"
                        done

                        # app services: patch the new image tag, then apply
                        for s in $SERVICES; do
                            m=$(find_manifest $s)
                            [ -n "$m" ] || { echo "ERROR: no manifest.yaml for $s"; exit 1; }
                            sed "s|image:.*|image: $DOCKER_USER/$s:$IMAGE_TAG|" "$m" | kubectl apply -f -
                        done

                        # wait for rollouts, roll back on failure
                        for s in $SERVICES; do
                            kubectl rollout status deployment/$s -n $K8S_NAMESPACE --timeout=300s || {
                                kubectl rollout undo deployment/$s -n $K8S_NAMESPACE
                                exit 1
                            }
                        done

                        kubectl get pods,svc -n $K8S_NAMESPACE
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''#!/bin/bash
                for s in $SERVICES; do docker rmi $DOCKER_USER/$s:$IMAGE_TAG || true; done
                rm -f kubeconfig-* || true
            '''
        }
        success { echo "Build ${BUILD_NUMBER} OK, image tag ${IMAGE_TAG}" }
        failure { echo "Build ${BUILD_NUMBER} FAILED, check the failed stage log" }
    }
}

pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_BAD_RELEASE',
            defaultValue: false,
            description: 'Deploy a faulty image to demonstrate automatic rollback'
        )
    }

    environment {
        IMAGE_NAME = 'afif729/hospital-appointment-portal'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (params.SIMULATE_BAD_RELEASE) {
                        bat 'docker build -f Dockerfile.faulty -t %IMAGE_NAME%:faulty .'
                    } else {
                        bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
                    }
                }
            }
        }

        stage('Test Application') {
            steps {
                script {
                    def testTag = params.SIMULATE_BAD_RELEASE ? 'faulty' : env.IMAGE_TAG

                    bat """
                        @echo off
                        docker rm -f hospital-ci-test 2>NUL
                        docker run -d --name hospital-ci-test %IMAGE_NAME%:${testTag}
                        if errorlevel 1 exit /b 1

                        docker exec hospital-ci-test sh -c "wget -q -O - http://127.0.0.1/ | grep -q CarePoint"
                        if errorlevel 1 (
                            docker logs hospital-ci-test
                            exit /b 1
                        )

                        echo Homepage test passed.
                    """
                }
            }
            post {
                always {
                    bat 'docker rm -f hospital-ci-test 2>NUL'
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    def pushTag = params.SIMULATE_BAD_RELEASE ? 'faulty' : env.IMAGE_TAG

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKERHUB_USER',
                            passwordVariable: 'DOCKERHUB_PASSWORD'
                        )
                    ]) {
                        bat """
                            @echo off
                            echo %DOCKERHUB_PASSWORD%| docker login -u %DOCKERHUB_USER% --password-stdin
                            if errorlevel 1 exit /b 1
                            docker push %IMAGE_NAME%:${pushTag}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-file',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    script {
                        withEnv(["KUBECONFIG=${env.KUBECONFIG_FILE}"]) {
                            def exists = bat(
                                returnStatus: true,
                                script: '@kubectl get deployment hospital-portal >NUL 2>&1'
                            )

                            if (exists != 0) {
                                bat 'kubectl apply -f k8s/deployment.yaml'
                                bat 'kubectl apply -f k8s/service.yaml'
                            }

                            // Capture the currently deployed image for recovery.
                            def previousImage = bat(
                                returnStdout: true,
                                script: '@kubectl get deployment hospital-portal -o jsonpath="{.spec.template.spec.containers[0].image}"'
                            ).trim()

                            def deployTag = params.SIMULATE_BAD_RELEASE ? 'faulty' : env.IMAGE_TAG
                            def newImage = "${env.IMAGE_NAME}:${deployTag}"

                            try {
                                bat "kubectl set image deployment/hospital-portal hospital-portal=${newImage}"

                                bat 'kubectl rollout status deployment/hospital-portal --timeout=90s'

                                // Confirm that three replicas are available.
                                bat 'kubectl get deployment hospital-portal'
                                bat 'kubectl get pods -l app=hospital-portal -o wide'
                                bat 'kubectl get service hospital-portal-service'
                                bat 'kubectl get endpoints hospital-portal-service'

                                // Smoke-test the live Service from inside the cluster.
                                bat '''
                                    kubectl run hospital-smoke-test --image=busybox:1.36 --restart=Never --rm -i --command -- sh -c "wget -q -O - http://hospital-portal-service/ | grep -q CarePoint"
                                '''

                                echo "Deployment and smoke test succeeded."
                            } catch (err) {
                                echo "Deployment failed. Restoring the previous image: ${previousImage}"

                                // Restore the exact previously active image.
                                bat(
                                    returnStatus: true,
                                    script: "kubectl set image deployment/hospital-portal hospital-portal=${previousImage}"
                                )

                                // Wait for the previous stable release to recover.
                                def rollbackStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout status deployment/hospital-portal --timeout=180s'
                                )

                                bat 'kubectl get pods -l app=hospital-portal -o wide'
                                bat 'kubectl get service hospital-portal-service'

                                if (rollbackStatus != 0) {
                                    error('Deployment failed and automatic recovery needs manual attention.')
                                }

                                error('New release failed. Previous stable image restored successfully.')
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Hospital portal CI/CD pipeline completed.'
        }
    }
}
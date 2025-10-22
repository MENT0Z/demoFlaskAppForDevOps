pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'ment00x'          // your Docker Hub username
        APP_NAME = 'flask'                   // app name
        KUBE_SERVICE = 'flask-service'      // Kubernetes service name
        // Set this to point to your minikube kubeconfig file location:
        KUBECONFIG = 'C:\Users\Madan Raj Upadhyay\.kube\config'  
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/MENT0Z/demoFlaskAppForDevOps.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t %DOCKER_HUB_USER%/flask-jenkins:v1 ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    bat """
                        echo %PASS% | docker login -u %USER% --password-stdin
                        docker push %DOCKER_HUB_USER%/flask-jenkins:v1
                    """
                }
            }
        }

        stage('Verify Kubernetes Access') {
            steps {
                bat 'kubectl cluster-info'
            }
        }

        stage('Determine Live Version') {
            steps {
                script {
                    def LIVE_VERSION = bat(script: 'kubectl get svc %KUBE_SERVICE% -o jsonpath="{.spec.selector.version}"', returnStdout: true).trim()
                    def NEXT_VERSION = LIVE_VERSION == 'blue' ? 'green' : 'blue'
                    env.LIVE_VERSION = LIVE_VERSION
                    env.NEXT_VERSION = NEXT_VERSION
                    echo "Live version is: ${LIVE_VERSION}"
                    echo "Next version to deploy: ${NEXT_VERSION}"
                }
            }
        }

        stage('Deploy Next Version') {
            steps {
                // Create deployment YAML dynamically
                bat """
                echo apiVersion: apps/v1 > k8s\\deployment-%NEXT_VERSION%.yaml
                echo kind: Deployment >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo metadata: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo   name: %APP_NAME%-%NEXT_VERSION% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo   labels: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo     app: %APP_NAME% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo     version: %NEXT_VERSION% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo spec: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo   replicas: 2 >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo   selector: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo     matchLabels: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo       app: %APP_NAME% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo       version: %NEXT_VERSION% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo   template: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo     metadata: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo       labels: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo         app: %APP_NAME% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo         version: %NEXT_VERSION% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo     spec: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo       containers: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo       - name: %APP_NAME% >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo         image: %DOCKER_HUB_USER%/flask-jenkins:v1 >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo         ports: >> k8s\\deployment-%NEXT_VERSION%.yaml
                echo         - containerPort: 5000 >> k8s\\deployment-%NEXT_VERSION%.yaml

                kubectl apply -f k8s\\deployment-%NEXT_VERSION%.yaml
                kubectl rollout status deployment/%APP_NAME%-%NEXT_VERSION%
                """
            }
        }

        stage('Smoke Test') {
            steps {
                bat """
                FOR /F %%i IN ('kubectl get svc %KUBE_SERVICE% -o jsonpath="{.status.loadBalancer.ingress[0].ip}"') DO SET SERVICE_IP=%%i
                curl -f http://%SERVICE_IP% || exit /b 1
                """
            }
        }

        stage('Switch Traffic') {
            steps {
                bat """
                kubectl patch service %KUBE_SERVICE% -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"%APP_NAME%\\",\\"version\\":\\"%NEXT_VERSION%\"}}}"
                """
            }
        }

        stage('Scale Down Old Version') {
            steps {
                bat 'kubectl scale deployment/%APP_NAME%-%LIVE_VERSION% --replicas=0 || exit /b 0'
            }
        }
    }

    post {
        failure {
            steps {
                bat """
                kubectl patch service %KUBE_SERVICE% -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"%APP_NAME%\\",\\"version\\":\\"%LIVE_VERSION%\"}}}"
                """
            }
        }
    }
}

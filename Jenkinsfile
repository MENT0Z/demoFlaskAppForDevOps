pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'ment00x'
        APP_NAME = 'flask'
        KUBE_SERVICE = 'flask-service'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/MENT0Z/demoFlaskAppForDevOps.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_HUB_USER/flask-jenkins:v1 .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                 usernameVariable: 'USER',
                                                 passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh 'docker push $DOCKER_HUB_USER/flask-jenkins:v1'
                }
            }
        }

        stage('Determine Live Version') {
            steps {
                script {
                    LIVE_VERSION = sh(script: "kubectl get svc $KUBE_SERVICE -o jsonpath='{.spec.selector.version}'", returnStdout: true).trim()
                    NEXT_VERSION = LIVE_VERSION == 'blue' ? 'green' : 'blue'
                    echo "Live: $LIVE_VERSION, Next: $NEXT_VERSION"
                }
            }
        }

        stage('Deploy Next Version') {
            steps {
                sh """
                cat <<EOF > k8s/deployment-${NEXT_VERSION}.yaml
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: $APP_NAME-${NEXT_VERSION}
                  labels:
                    app: $APP_NAME
                    version: $NEXT_VERSION
                spec:
                  replicas: 2
                  selector:
                    matchLabels:
                      app: $APP_NAME
                      version: $NEXT_VERSION
                  template:
                    metadata:
                      labels:
                        app: $APP_NAME
                        version: $NEXT_VERSION
                    spec:
                      containers:
                      - name: $APP_NAME
                        image: $DOCKER_HUB_USER/flask-jenkins:v1
                        ports:
                        - containerPort: 5000
                EOF

                kubectl apply -f k8s/deployment-${NEXT_VERSION}.yaml
                kubectl rollout status deployment/$APP_NAME-${NEXT_VERSION}
                """
            }
        }

        stage('Smoke Test') {
            steps {
                sh """
                SERVICE_IP=\$(kubectl get svc $KUBE_SERVICE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                curl -f http://\$SERVICE_IP || exit 1
                """
            }
        }

        stage('Switch Traffic') {
            steps {
                sh """
                kubectl patch service $KUBE_SERVICE -p '{"spec":{"selector":{"app":"'$APP_NAME'","version":"'$NEXT_VERSION'"}}}'
                """
            }
        }

        stage('Scale Down Old Version') {
            steps {
                sh 'kubectl scale deployment/$APP_NAME-$LIVE_VERSION --replicas=0 || true'
            }
        }
    }

    post {
        failure {
            script {
                sh 'kubectl patch service $KUBE_SERVICE -p \'{"spec":{"selector":{"app":"'$APP_NAME'","version":"'$LIVE_VERSION'"}}}\''
            }
        }
    }
}

pipeline {
    agent {
        kubernetes {
            label 'openshift-jenkins-pipeline'
            defaultContainer 'maven'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: openshift-jenkins-agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: maven
    image: registry.access.redhat.com/ubi8/ubi:8.8
    command:
    - cat
    tty: true
  - name: oc-cli
    image: registry.access.redhat.com/openshift4/ose-cli:latest
    command:
    - cat
    tty: true
"""
        }
    }

    environment {
        PROJECT = 'auto'
        APP_NAME = 'sample-app-jenkins-new'
        GIT_REPO = 'https://github.com/shrijandra/openshift-jenkins-cicd.git'
        GIT_CRED = 'github-cred'
    }

    stages {

        stage('Checkout Code') {
            steps {
                container('maven') {
                    git branch: 'main', url: "${env.GIT_REPO}", credentialsId: "${env.GIT_CRED}"
                }
            }
        }

        stage('Build with Maven (Optional)') {
            steps {
                container('maven') {
                    sh 'mvn -B clean package -DskipTests'
                }
            }
        }

        stage('Create BuildConfig') {
            steps {
                container('oc-cli') {
                    script {
                        echo "Creating OpenShift binary BuildConfig..."
                        openshift.withCluster() {
                            openshift.withProject("${env.PROJECT}") {
                                // Delete existing BC if it exists
                                sh "oc delete bc ${env.APP_NAME} --ignore-not-found"

                                // Create binary BuildConfig with S2I using OpenJDK 17
                                sh """
                                    oc new-build --name=${env.APP_NAME} \
                                    --image-stream=ubi8-openjdk-17:latest \
                                    --binary=true \
                                    --strategy=source \
                                    --to=${env.APP_NAME}:latest
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Start S2I Binary Build') {
            steps {
                container('oc-cli') {
                    script {
                        echo "Starting S2I binary build..."
                        openshift.withCluster() {
                            openshift.withProject("${env.PROJECT}") {
                                // Prepare directory with built JAR for binary build
                                sh "mkdir -p ocp/deployments"
                                sh "cp target/*.jar ocp/deployments/ || true"

                                // Start S2I build
                                sh "oc start-build ${env.APP_NAME} --from-dir=ocp --wait --follow"
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy App') {
            steps {
                container('oc-cli') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject("${env.PROJECT}") {
                                echo "Deploying app..."
                                // Delete DC if exists to avoid conflicts
                                sh "oc delete dc ${env.APP_NAME} --ignore-not-found"

                                // Create a new app from the built image
                                sh "oc new-app ${env.APP_NAME}:latest --name=${env.APP_NAME}"

                                // Expose the service
                                sh "oc expose svc/${env.APP_NAME} || true"
                            }
                        }
                    }
                }
            }
        }
    }
}

pipeline {

    // Use a Kubernetes agent pod with Maven + Java + oc tools
    agent {
        kubernetes {
            cloud 'openshift'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: registry.redhat.io/openshift4/ose-jenkins-agent-maven
    command:
    - cat
    tty: true
'''
        }
    }

    environment {
        APP_NAME = "sample-app-jenkins-new"
        GIT_URL  = "https://github.com/shrijandra/openshift-jenkins-cicd.git"
        GIT_CRED = "github-cred"
        JAR_FILE = "openshiftjenkins-0.0.1-SNAPSHOT.jar"
    }

    stages {

        stage('Clone Repository') {
            steps {
                container('maven') {
                    git branch: 'main',
                        credentialsId: env.GIT_CRED,
                        url: env.GIT_URL
                }
            }
        }

        stage('Build App') {
            steps {
                container('maven') {
                    sh "mvn clean install"
                }
            }
        }

        stage('Prepare Artifacts') {
            steps {
                container('maven') {
                    sh """
                        rm -rf ocp && mkdir -p ocp/deployments
                        cp target/${JAR_FILE} ocp/deployments/
                    """
                }
            }
        }

        stage('Create Image Builder (BC)') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject() {
                            return !openshift.selector("bc", env.APP_NAME).exists()
                        }
                    }
                }
            }
            steps {
                container('maven') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                openshift.newBuild(
                                    "--name=${env.APP_NAME}",
                                    "--image-stream=openjdk18-openshift:1.14-3",
                                    "--binary=true"
                                )
                            }
                        }
                    }
                }
            }
        }

        stage('Build Image (Binary Build)') {
            steps {
                container('maven') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                openshift.selector("bc", env.APP_NAME)
                                    .startBuild("--from-dir=./ocp", "--follow", "--wait=true")
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to OpenShift') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject() {
                            return !openshift.selector("dc", env.APP_NAME).exists()
                        }
                    }
                }
            }
            steps {
                container('maven') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                def app = openshift.newApp(env.APP_NAME, "--as-deployment-config")
                                app.narrow("svc").expose()
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Build Failed!"
        }
        success {
            echo "✅ Application ${env.APP_NAME} deployed successfully on OpenShift!"
        }
    }
}


pipeline {
    agent {
        kubernetes {
            label 'maven-agent'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: image-registry.openshift-image-registry.svc:5000/<your-project>/my-custom-maven-agent:latest
    command:
    - cat
    tty: true
"""
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }
    

        stage('Create Image Builder') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject() {
                            !openshift.selector("bc", "sample-app-jenkins-new").exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.newBuild(
                                "--name=sample-app-jenkins-new",
                                "--image-stream=openjdk18-openshift:1.14-3",
                                "--binary=true"
                            )
                        }
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh "rm -rf ocp && mkdir -p ocp/deployments"
                sh "cp target/*.jar ocp/deployments/"

                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.selector("bc", "sample-app-jenkins-new")
                                .startBuild("--from-dir=./ocp", "--follow", "--wait=true")
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject() {
                            !openshift.selector("dc", "sample-app-jenkins-new").exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def app = openshift.newApp("sample-app-jenkins-new", "--as-deployment-config")
                            app.narrow("svc").expose()
                        }
                    }
                }
            }
        }
    }
}

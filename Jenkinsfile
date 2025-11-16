pipeline {
    agent {
        kubernetes {
            label "maven"
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: maven-agent
spec:
  containers:
  - name: maven
    image: maven:3.9.6-eclipse-temurin-17
    command:
    - cat
    tty: true
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  initContainers:
  - name: install-oc
    image: registry.access.redhat.com/ubi8/ubi
    command:
      - sh
      - -c
      - |
        curl -L https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz \
        | tar -xz && mv oc kubectl /usr/local/bin/
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    stages {

        stage('Build App') {
            steps {
                container('maven') {
                    git branch: 'main', credentialsId: 'github-cred', url: 'https://github.com/kuldeepsingh99/openshift-jenkins-cicd.git'
                    script {
                        def pom = readMavenPom file: 'pom.xml'
                        version = pom.version
                    }
                    sh "mvn -B clean install"
                }
            }
        }

        stage('Create Image Builder') {
            steps {
                container('maven') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                if (!openshift.selector("bc", "sample-app-jenkins-new").exists()) {
                                    openshift.newBuild("--name=sample-app-jenkins-new",
                                        "--image-stream=openjdk18-openshift:1.14-3",
                                        "--binary=true")
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                container('maven') {
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
        }

        stage('Deploy') {
            steps {
                container('maven') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                if (!openshift.selector('dc', 'sample-app-jenkins-new').exists()) {
                                    def app = openshift.newApp("sample-app-jenkins-new", "--as-deployment-config")
                                    app.narrow("svc").expose()
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

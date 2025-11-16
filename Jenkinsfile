pipeline {
    agent {
        kubernetes {
            // Dynamic pod template for this pipeline
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    pipeline: maven-oc
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

    environment {
        APP_NAME = "sample-app-jenkins-new"
        PROJECT = "auto"           // replace with your OpenShift project
        IMAGE_STREAM = "openjdk-17"        // S2I builder image
    }

    stages {

        stage('Checkout Code') {
            steps {
                container('maven') {
                    git branch: 'main', credentialsId: 'github-cred', url: 'https://github.com/kuldeepsingh99/openshift-jenkins-cicd.git'
                }
            }
        }

        stage('Maven Build') {
            steps {
                container('maven') {
                    sh "mvn -B clean package -DskipTests"
                }
            }
        }

        stage('OpenShift Login') {
            steps {
                container('maven') {
                    sh '''
                        oc login --token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
                                 --server=https://kubernetes.default.svc
                        oc project $PROJECT
                    '''
                }
            }
        }

        stage('Create BuildConfig if Missing') {
            steps {
                container('maven') {
                    sh """
                        if ! oc get bc $APP_NAME >/dev/null 2>&1; then
                            echo "Creating S2I BuildConfig..."
                            oc new-build $IMAGE_STREAM~. --name=$APP_NAME --binary=true
                        fi
                    """
                }
            }
        }

        stage('Start S2I Binary Build') {
            steps {
                container('maven') {
                    sh """
                        mkdir -p ocp
                        cp target/*.jar ocp/
                        oc start-build $APP_NAME --from-dir=ocp --follow --wait=true
                    """
                }
            }
        }

        stage('Deploy to OpenShift') {
            steps {
                container('maven') {
                    sh """
                        if ! oc get deployment $APP_NAME >/dev/null 2>&1; then
                            oc create deployment $APP_NAME \
                                --image=image-registry.openshift-image-registry.svc:5000/$PROJECT/$APP_NAME:latest
                            oc expose deployment $APP_NAME
                            oc expose service $APP_NAME
                        else
                            oc set image deployment/$APP_NAME \
                                $APP_NAME=image-registry.openshift-image-registry.svc:5000/$PROJECT/$APP_NAME:latest
                            oc rollout restart deployment/$APP_NAME
                        fi
                    """
                }
            }
        }
    }
}

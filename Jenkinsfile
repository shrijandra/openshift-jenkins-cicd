pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    pipeline: maven-oc
spec:
  containers:
  - name: maven
    image: default-route-openshift-image-registry.apps-crc.testing/openshift/jenkins-agent-base:latest
    command:
      - cat
    tty: true
"""
        }
    }

    environment {
        APP_NAME = "sample-app-jenkins-new"
        PROJECT = "auto"
        IMAGE_STREAM = "openjdk-17"  // adjust to your OpenShift S2I image if needed
    }

    stages {

        stage('Checkout Code') {
            steps {
                container('maven') {
                    git branch: 'main',
                        url: 'https://github.com/shrijandra/openshift-jenkins-cicd.git',
                        credentialsId: 'github-cred'
                }
            }
        }

        stage('Install Maven (if needed)') {
            steps {
                container('maven') {
                    sh '''
                        if ! command -v mvn >/dev/null 2>&1; then
                            echo "Installing Maven..."
                            MAVEN_HOME=$HOME/apache-maven-3.9.6
                            mkdir -p $MAVEN_HOME
                            curl -L --fail -o /tmp/apache-maven-3.9.6-bin.tar.gz \
                                https://archive.apache.org/dist/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
                            tar -xzf /tmp/apache-maven-3.9.6-bin.tar.gz -C $HOME
                            export PATH=$MAVEN_HOME/bin:$PATH
                        fi
                        mvn -version
                    '''
                }
            }
        }


        stage('Maven Build') {
            steps {
                container('maven') {
                    sh 'mvn -B clean package -DskipTests'
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
                        oc version
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

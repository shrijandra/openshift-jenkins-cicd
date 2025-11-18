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
  serviceAccountName: jenkins
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
        PATH_ADD = "/home/jenkins/apache-maven-3.9.6/bin:/home/jenkins/bin"
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
                            
                        fi
                        # Add Maven to PATH and build
                        export PATH=$HOME/apache-maven-3.9.6/bin:$PATH
                        
                        # Install OC CLI if not found
                        
                        if ! command -v oc >/dev/null 2>&1; then
                            echo "Installing OpenShift OC CLI..."
                            mkdir -p $HOME/bin
                            curl -L --fail https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz \
                                | tar -xz -C $HOME/bin
                            chmod +x $HOME/bin/oc
                        fi

                        export PATH=$MAVEN_HOME/bin:$PATH
                        
                        mvn -version
                        oc version --client
                        mvn -B clean package -DskipTests

                        
                    '''
                }
            }
        }

        stage('OpenShift Login') {
            steps {
                container('maven') {
                    sh '''
                        export PATH=$HOME/bin:$PATH
                        oc login --token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
                                 --server=https://kubernetes.default.svc \
                                 --insecure-skip-tls-verify
                        oc project $PROJECT
                        oc version
                    '''
                }
            }
        }

        stage('Create BuildConfig') {
            steps {
                container('maven') {
                  sh '''
                    export PATH=${PATH_ADD}:$PATH

            # Delete existing BC (optional) to ensure created state - safe for CI
                    oc delete bc ${APP_NAME} --ignore-not-found

            # Create a binary S2I BuildConfig using ubi8-openjdk-17 image stream
                    oc new-build --name=${APP_NAME} \
                                 --image-stream=ubi8-openjdk-17:latest \
                                 --binary=true \
                                 --strategy=source \
                                 --to=${APP_NAME}:latest
                  '''
                }
            }
        }


        stage('Start S2I Binary Build') {
            steps {
                container('maven') {
                    sh """
                        export PATH=${PATH_ADD}:$PATH
                        
                        # prepare binary dir for S2I
                        rm -rf ocp || true
                        mkdir -p ocp/deployments

                        # if a jar exists (built above), send it; else send source so builder runs maven
                        if ls target/*.jar >/dev/null 2>&1; then
                          cp target/*.jar ocp/deployments/
                        else
                          cp -r . ocp/
                        fi

                        # start the binary build and wait for completion
                        oc start-build ${APP_NAME} --from-dir=ocp --follow --wait
                      '''
                }
            }
        }

        stage('Deploy to OpenShift') {
            steps {
                container('maven') {
                    sh """
                        export PATH=${PATH_ADD}:$PATH
                    
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
                        # Expose service/route
                        oc expose svc/${APP_NAME} --port=8080 || true

                        echo "Route (if created):"
                        oc get route ${APP_NAME} -o yaml || true
                     """
                 }
             }
         }
    }
}

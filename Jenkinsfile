pipeline {

    agent {
        kubernetes {
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
     app.kubernetes.io/name: jenkins-build
     app.kubernetes.io/component: jenkins-build
     app.kubernetes.io/version: "1"
spec:
  volumes:
   - name: secret-volume
     secret:
       secretName: pks-cicd  
  hostAliases:
  - ip: 172.168.21.15
    hostnames:
    - "harbor.icdscore.local"
  containers:
  - name: k8s
    image: mcdstanzu/tbsdocker:latest
    command:
    - sleep
    env:
      - name: KUBECONFIG
        value: "/tmp/config/jenkins-sa"
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/tmp/config"    
    args:
    - infinity
"""
        }
    }

    environment {
        APP_NAME="${env.APP_NAME}"
    }

    stages {

        stage('Fetch from GitHub') {
            steps {
                dir("app"){
                    git(
                        poll: true,
                        changelog: true,
                        branch: "main",
                        credentialsId: "git-jenkins",
                        url: "git@github.com:AkhilCh308/${APP_NAME}.git"
                    )
                    sh 'git rev-parse HEAD > git-commit.txt'
                }
            }
        }

        stage('Create Image') {
            steps {
                container('k8s') {
                    sh '''#!/bin/sh -e
                        export GIT_COMMIT=$(cat app/git-commit.txt)
                        kp image save ${APP_NAME} \
                            --git git@github.com:AkhilCh308/${APP_NAME}.git \
                            -t harbor.icdscore.local/akhil/${APP_NAME} \
                            --env BP_GRADLE_BUILD_ARGUMENTS='--no-daemon build' \
                            --git-revision ${GIT_COMMIT} -w
                    '''
                }
            }
        }

        stage('Update Deployment Manifest'){
            steps {
                container('k8s'){
                    dir("gitops"){
                        git(
                            poll: false,
                            changelog: false,
                            branch: "master",
                            credentialsId: "git-jenkins",
                            url: "git@github.com:jeffellin/spring-petclinic-gitops.git"
                        )
                    }
                    sshagent(['git-jenkins']) {   
                        sh '''#!/bin/sh -e
                        
                        kubectl get image ${APP_NAME} -o json | jq -r .status.latestImage >> containerversion.txt
                        export CONTAINER_VERSION=$(cat containerversion.txt)
                        cd gitops/app
                        kustomize edit set image ${APP_NAME}=${CONTAINER_VERSION}
                        git config --global user.name "jenkins CI"
                        git config --global user.email "none@none.com"
                        git add .
                        git diff-index --quiet HEAD || git commit -m "update by ci"
                        mkdir -p ~/.ssh
                        ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts
                        git pull -r origin master
                        git push --set-upstream origin master
                        '''
                    }
                }  
            }
        }
    }
}

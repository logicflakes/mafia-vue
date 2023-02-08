import groovy.json.JsonSlurper
pipeline {
    agent {
        kubernetes {
            defaultContainer 'alpine'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.7.0-debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
    - name: shared-data
      mountPath: /shared-data
  - name: alpine
    image: alpine
    imagePullPolicy: Always
    command:
    - /bin/cat
    tty: true
    volumeMounts:
    - name: shared-data
      mountPath: /shared-data
"""
        }
    }
    environment {
        IMAGE_NAMESPACE="registry.test.relizahub.com/f3b39816-e827-4882-ad37-835c011134bb-public"
        IMAGE_NAME="mafia-vue"
        RELIZA_API=credentials('RELIZA_API')
        container="kube"
        BITBUCKET_API_URL="https://api.bitbucket.org/2.0/repositories/relizar/mafia-vue"
        BITBUCKET_TOKEN=credentials('BITBUCKET_TOKEN')
    }
    stages {
        stage('Print push payload') {
            steps {
                script {
                    sh 'printenv'
                }
            }
        }
        stage('Build with Kaniko') {
            steps {
                script {
                    sh 'apk add git'
                    sh 'git config --global --add safe.directory \'*\''
                    setPrDetailsOnEnv()
                    env.COMMIT_TIME = sh(script: 'git log -1 --date=iso-strict --pretty="%ad"', returnStdout: true).trim()
                    withReliza(projectId: '094c08d5-4581-4555-9ad9-81e93d2b47f1', uri: 'https://test.relizahub.com') {
                        if (env.LATEST_COMMIT) {
                            env.COMMIT_LIST = getCommitListWithLatest()
                        } else {
                            env.COMMIT_LIST = getCommitListNoLatest()
                        }
                        if (!env.LATEST_COMMIT || env.COMMIT_LIST) {
                            try {
                                container(name: 'kaniko', shell: '/busybox/sh') {
                                    withCredentials([file(credentialsId: 'docker-credentials', variable: 'DOCKER_CONFIG_JSON')]) {
                                        withEnv(['PATH+EXTRA=/busybox']) {
                                            sh '''#!/busybox/sh
                                                cp $DOCKER_CONFIG_JSON /kaniko/.docker/config.json
                                                /kaniko/executor --context `pwd` --destination "$IMAGE_NAMESPACE/$IMAGE_NAME:latest" --digest-file=/shared-data/termination-log --build-arg CI_ENV=Jenkins --build-arg GIT_COMMIT=$GIT_COMMIT --build-arg GIT_BRANCH=$GIT_BRANCH --build-arg VERSION=$VERSION --cache=true
                                            '''
                                        }
                                    }
                                }
                                env.SHA_256 = sh(script: 'cat /shared-data/termination-log', returnStdout: true).trim()
                                echo "SHA 256 digest of our container = ${env.SHA_256}"
                            } catch (Exception e) {
                                env.STATUS = 'rejected'
                                echo 'FAILED BUILD: ' + e.toString()
                                currentBuild.result = 'FAILURE'
                            }
                            addRelizaRelease(artId: "$IMAGE_NAMESPACE/$IMAGE_NAME", artType: "Docker", useCommitList: 'true')
                        } else {
                            echo 'Repeated build, skipping push'
                        }
                    }
                }
            }
        }
    }
}

String getCommitListNoLatest() {
  if (env.GIT_PREVIOUS_SUCCESSFUL_COMMIT) {
    return sh(script: 'git log $GIT_PREVIOUS_SUCCESSFUL_COMMIT..$GIT_COMMIT --date=iso-strict --pretty="%H|||%ad|||%s" -- ./ | base64 -w 0', returnStdout: true).trim()
  } else {
    return sh(script: 'git log -1 --date=iso-strict --pretty="%H|||%ad|||%s" -- ./ | base64 -w 0', returnStdout: true).trim()
  }
}

String getCommitListWithLatest() {
  return sh(script: 'git log $LATEST_COMMIT..$GIT_COMMIT --date=iso-strict --pretty="%H|||%ad|||%s" -- ./ | base64 -w 0', returnStdout: true).trim()
}

def setPrDetailsOnEnv(){
    sh 'apk add jq'
    sh 'apk add curl'
   
    env.commitPr = sh(script: "curl --request GET --url '$BITBUCKET_API_URL/commit/$GIT_COMMIT/pullrequests' --header 'Accept: application/json' --header 'Authorization: Bearer $BITBUCKET_TOKEN'", returnStdout: true)
    sh 'echo $commitPr'
    def prid = sh(script: 'echo $commitPr | jq -r ".values[0].id"', returnStdout: true)
    echo prid
    if(prid != 'null'){
        echo "prid=$prid not equal null"
        def prData = sh(script: "curl --request GET --url '$BITBUCKET_API_URL/pullrequests/$prid' --header 'Accept: application/json' --header 'Authorization: Bearer $BITBUCKET_TOKEN'", returnStdout: true)
        echo prData
    }
    
}
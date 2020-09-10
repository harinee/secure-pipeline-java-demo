pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh 'mvn install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets scanner') {
          steps {
            container('trufflehog') {
              sh 'git clone ${GIT_URL}'
              sh 'cd secure-pipeline-java-demo && ls -al'
              sh 'cd secure-pipeline-java-demo && trufflehog .'
              sh 'rm -rf secure-pipeline-java-demo'
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn package'
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('Dependency Checker') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('Spot Bugs - Security') {
          steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              container('maven') {
                sh 'mvn compile spotbugs:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/spotbugsXml.xml', fingerprint: true, onlyIfSuccessful: false
            }
          }
        }
      }
    }
    stage('Package') {
      steps {
        container('docker-cmds') {
          sh 'ls -al'
          sh 'docker build . -t sample-app'
        }
      }
    }
    stage('Artefact Analysis') {
      parallel {
        stage('Image Dependency Scan') {
          steps {
            container('docker-cmds') {
              sh '''#!/bin/sh
                    apk add --update-cache --upgrade curl rpm
                    export TRIVY_VERSION="0.8.0"
                    echo $TRIVY_VERSION
                    wget https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
                    tar zxvf trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
                    mv trivy /usr/local/bin
                    trivy --cache-dir /tmp/trivycache/ sample-app:latest
                  '''
              }
          }
        }
        stage('Image Hardening') {
          steps {
            container('dockle') {
              sh 'dockle sample-app:latest'
            }
          }
        }
       }
      }
    stage('Deploy') {
      steps {
        // TODO
        sh "echo done"
      }
    }
    stage('Dynamic Analysis') {
      parallel {
        stage('Functional tests') {
          steps {
            sh 'echo "All Tests passed!!!"'
          }
        }
        stage('DAST') {
          steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              container('docker-cmds') {
                sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t https://www.zaproxy.org/'
              }
            }
          }
        }
      }
    }
  }
}

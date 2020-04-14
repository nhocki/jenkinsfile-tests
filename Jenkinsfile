pipeline {
  agent {
    label 'build_node_codebuild'
  }

  stages {
    stage('platform') {
      parallel {
        stage('running shellcheck') {
          agent {
            docker {
              image 'koalaman/shellcheck-alpine'
              args "-v ${WORKSPACE}:/mnt"
            }
          }

          steps {
            githubNotify context: 'shellcheck', status: 'PENDING', description: ''
            script {
              try{
                sh '''
                find /mnt -name '*.sh' | xargs -r shellcheck
                '''
                githubNotify context: 'shellcheck', status: 'SUCCESS', description: ''
              }
              catch (err) {
                githubNotify context: 'shellcheck', status: 'FAILURE', description: ''
                throw(err)
              }
            }
          }
        }

        stage('make some-binary') {
          steps {
            script {
              if (env.BRANCH_NAME == "master") {
                githubNotify context: 'cmd/some-binary', status: 'PENDING', description: ''
                script {
                  try{
                    awsCodeBuild projectName: 'platform-make',
                    region: 'us-west-1',
                    credentialsType: 'keys',
                    credentialsId: 'whatever',
                    sourceVersion: env.GIT_COMMIT,
                    sourceControlType: 'project',
                    envVariables: "[{SERVICE, ../}, {TARGET, some-binary}]"
                  }
                  catch (err) {
                    githubNotify context: 'cmd/some-binary', status: 'FAILURE', description: ''
                    throw(err)
                  }
                }
                githubNotify context: 'cmd/some-binary', status: 'SUCCESS', description: ''
              }
            }
          }
        }

        stage('make libs') {
          steps {
            githubNotify context: 'libraries/tests', status: 'PENDING', description: ''
            script {
              try{
                awsCodeBuild projectName: 'platform-make',
                region: 'us-west-1',
                credentialsType: 'keys',
                credentialsId: 'whatever',
                sourceVersion: env.GIT_COMMIT,
                sourceControlType: 'project',
                envVariables: "[{SERVICE, ../}, {TARGET, libs}]"
              }
              catch (err) {
                githubNotify context: 'libraries/tests', status: 'FAILURE', description: ''
                throw(err)
              }
            }
            githubNotify context: 'libraries/tests', status: 'SUCCESS', description: ''
          }
        }

        stage('run services pipelines') {
          steps {
            script {
              parallel buildServices()
            }
          }
        }
      }
    }
  }
}

// buildServices returns a list of tests for each service.
def buildServices() {
  def services = [:]
  files = sh(script: 'find ./services/ -type f -name \'Jenkinsfile\' | sed -r \'s|/[^/]+$||\' |sort |uniq', returnStdout: true).split()
  files.each { item ->
    services.put("${item.split("/").last()}", prepareService(item))
  }
  return services
}

def prepareService(String service) {
  return {
    load("${service}/Jenkinsfile")
  }
}

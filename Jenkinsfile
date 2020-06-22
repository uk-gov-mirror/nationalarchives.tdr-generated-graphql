library("tdr-jenkinslib")

def versionBumpBranch = "version-bump-${BUILD_NUMBER}"

pipeline {
  agent none

  parameters {
    choice(name: "STAGE", choices: ["intg", "staging", "prod"], description: "The stage you are deploying the schema to")
    text(name: "SCHEMA", defaultValue: "")
  }
  stages {
    stage("Create version bump branch") {
      agent {
        label "master"
      }
      steps {
        script {
          tdr.configureJenkinsGitUser()
        }
        sh "git checkout -b ${versionBumpBranch}"
        script {
          tdr.pushGitHubBranch("${versionBumpBranch}")
        }
      }
    }
    stage("Deployment") {
      parallel {
        stage("Deploy to npm") {
          agent {
            ecs {
              inheritFrom "npm"
            }
          }
          stages {
            stage ("Checkout and track version bump branch") {
              steps {
                script {
                  tdr.configureJenkinsGitUser()
                }
                sshagent(['github-jenkins']) {
                  sh "git checkout -b ${versionBumpBranch} --track origin/${versionBumpBranch}"
                }
              }
            }
            stage("Update npm version") {
              steps {
                sh "mkdir -p src/main/resources"
                sh "echo '${params.SCHEMA.trim()}' > src/main/resources/schema.graphql"
                dir("ts") {
                  sh 'npm ci'
                  sh 'npm run codegen'
                  sh 'npm run build'

                  //commits to local branch
                  sshagent(['github-jenkins']) {
                    sh "npm version patch"
                  }

                  withCredentials([string(credentialsId: 'npm-login', variable: 'LOGIN_TOKEN')]) {
                    sh "npm config set //registry.npmjs.org/:_authToken=$LOGIN_TOKEN"
                    sh 'npm publish --access public'
                  }
                }
              }
            }
            stage("Commit npm version bump changes to branch") {
              steps {
                script {
                  tdr.configureJenkinsGitUser()
                }
                sshagent(['github-jenkins']) {
                  sh "git pull"
                }
                script {
                  tdr.pushGitHubBranch(versionBumpBranch)
                }
              }
            }
          }
        }
        stage("Deploy to s3") {
          agent {
            ecs {
              inheritFrom "base"
              taskDefinitionOverride "arn:aws:ecs:eu-west-2:${env.MANAGEMENT_ACCOUNT}:task-definition/s3publish-${params.STAGE}:2"
            }
          }
          stages {
            stage("Checkout and track branch version bump branch") {
              steps {
                script {
                  tdr.configureJenkinsGitUser()
                }
                sshagent(['github-jenkins']) {
                  sh "git checkout -b ${versionBumpBranch} --track origin/${versionBumpBranch}"
                }
              }
            }
            stage("Update sbt release") {
              steps {
                sh "mkdir -p src/main/resources"
                sh "echo '${params.SCHEMA.trim()}' > src/main/resources/schema.graphql"

                //commits to origin branch
                sshagent(['github-jenkins']) {
                  sh "sbt 'release with-defaults'"
                }

                slackSend color: "good", message: "*GraphQL schema* :arrow_up: The generated GraphQL schema has been published", channel: "#bot-testing"
              }
            }
          }
        }
      }
    }
    stage("Create pull requests") {
      agent {
        label "master"
      }
      stages {
        stage("Create version bump pull request") {
          steps {
            script {
              tdr.createGitHubPullRequest(
                pullRequestTitle: "Version Bump from build number ${BUILD_NUMBER}",
                buildUrl: env.BUILD_URL,
                repo: "tdr-generated-graphql",
                branchToMergeTo: "master",
                branchToMerge: versionBumpBranch
              )
            }
          }
        }
      }
    }
  }
}

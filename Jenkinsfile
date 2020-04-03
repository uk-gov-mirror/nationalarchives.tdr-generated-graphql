pipeline {
    agent {
        label "master"
    }
    parameters {
        choice(name: "STAGE", choices: ["intg", "staging", "prod"], description: "The stage you are building the front end for")
        text(name: "SCHEMA", defaultValue: "")
    }
    stages {
        stage("Deploy") {
            parallel {
                stage("Deploy to npm") {
                    agent {
                        ecs {
                            inheritFrom "npm"
                        }
                    }
                    steps {
                        sh "mkdir -p src/main/resources"
                        sh "echo '${params.SCHEMA.trim()}' > src/main/resources/schema.graphql"
                        dir("ts"){
                            sh 'npm ci'
                            sh 'npm run codegen'
			    sh 'npm run build'
                            sh 'git config --global user.email tna-digital-archiving-jenkins@nationalarchives.gov.uk'
                            sh 'git config --global user.name tna-digital-archiving-jenkins'
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
                stage("Deploy to s3") {
                    agent {
                        ecs {
                            inheritFrom "base"
                            taskDefinitionOverride "arn:aws:ecs:eu-west-2:${env.MANAGEMENT_ACCOUNT}:task-definition/s3publish-${params.STAGE}:1"
                        }
                    }
                    steps {
                        script {
                            sh "mkdir -p src/main/resources"
                            sh "echo '${params.SCHEMA.trim()}' > src/main/resources/schema.graphql"
                            sshagent(['github-jenkins']) {
                                sh "git push --set-upstream origin ${env.GIT_LOCAL_BRANCH}"
                                sh 'git config --global user.email tna-digital-archiving-jenkins@nationalarchives.gov.uk'
                                sh 'git config --global user.name tna-digital-archiving-jenkins'
                                sh "sbt 'release with-defaults'"
                                slackSend color: "good", message: "*GraphQL schema* :arrow_up: The generated GraphQL schema has been published", channel: "#tdr-releases"
                            }

                        }
                    }
                }
            }
        }
    }
}

def getAccountNumberFromStage() {
    def stageToAccountMap = [
            "intg": env.INTG_ACCOUNT,
            "staging": env.STAGING_ACCOUNT,
            "prod": env.PROD_ACCOUNT
    ]

    return stageToAccountMap.get(params.STAGE)
}

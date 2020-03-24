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
                stage("Deploy to sonatype") {
                    agent {
                        ecs {
                            inheritFrom "base"
                            taskDefinitionOverride "arn:aws:ecs:eu-west-2:${env.MANAGEMENT_ACCOUNT}:task-definition/sonatype-${params.STAGE}:1"
                        }
                    }
                    steps {
                        script {
                            sh "mkdir -p src/main/resources"
                            sh "echo '${params.SCHEMA.trim()}' > src/main/resources/schema.graphql"
                            sh "aws s3 cp s3://tdr-secrets/keys/sonatype.key /home/jenkins/sonatype.key"
                            sh "aws s3 cp s3://tdr-secrets/keys/sonatype_credential /home/jenkins/.sbt/sonatype_credential"
                            withCredentials([string(credentialsId: 'sonatype-gpg-passphrase', variable: 'PGP_PASSPHRASE')]) {
                                sh 'gpg --batch --passphrase $PGP_PASSPHRASE --import /home/jenkins/sonatype.key'
                                sshagent(['github-jenkins']) {
                                    sh 'git push --set-upstream origin master'
                                    sh 'git config --global user.email tna-digital-archiving-jenkins@nationalarchives.gov.uk'
                                    sh 'git config --global user.name tna-digital-archiving-jenkins'
                                    sh "sbt +'release with-defaults'"
                                    slackSend color: "good", message: "*GraphQL schema* :arrow_up: The generated GraphQL schema has been published", channel: "#tdr-releases"
                                }
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

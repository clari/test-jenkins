def clariDeployEnv = 'steelix'
def clariDeployBranch = 'develop'
def sandboxName = 'default'
def deployEnv = 'development'
def gitTag = 'development-live'
def fixedClariDeployBranch = ''
def jenkinsTimeout = "40"

switch (clariDeployEnv) {
    case ~/production/:
        def deployEpoch = '2018/02/06'
        def rc = new org.deploy.Release(deployEpoch)
        def deployDate = rc.getBranch()
        fixedClariDeployBranch = "production-release-${deployDate}"
        deployEnv = 'production'
        gitTag = 'production-live'
        break
    case ~/sandbox/:
        fixedClariDeployBranch = "${clariDeployBranch}"
        deployEnv = "${clariDeployEnv}"
        gitTag = "${sandboxName}-live"
        break
    default:
        fixedClariDeployBranch = "${clariDeployBranch}"
        deployEnv = "${clariDeployEnv}"
        gitTag = "${clariDeployEnv}-live"
        break
}

def checkoutVar = [
    $class: 'GitSCM',
    branches: [
        [name: "*/${fixedClariDeployBranch}"]
    ],
    doGenerateSubmoduleConfigurations: false,
    extensions: [
         [$class: 'LocalBranch', localBranch: '**']
    ],
    submoduleCfg: [],
    userRemoteConfigs: [[
        credentialsId: 'clarius-deploy-key',
        name: 'origin',
        refspec: "+refs/heads/${fixedClariDeployBranch}:refs/remotes/origin/${fixedClariDeployBranch}",
        url: 'git@github.com:clari/dionysus.git'
    ]]
]

pipeline {
    agent {label 'staging-node'}

    options {
        ansiColor('xterm')
        timeout(time: jenkinsTimeout, activity: true, unit: 'MINUTES')
    }

    environment {
        GIT_BRANCH = "${fixedClariDeployBranch}"
        DIONYSUS_STAGE = "${deployEnv}"
        GIT_TAG = "${gitTag}"
        GIT_ENV = "${deployEnv}"
    }

    stages {
        // build stage
        stage('Build') {
            steps {
                milestone(ordinal: 1, label: 'initialize build')
                node('staging-node') {
                    checkout(checkoutVar)
                    script {
                        // def newBranch = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%H'").trim()
                        // checkoutVar.branches = [[name: newBranch]]

                        // def buildTimer = new org.deploy.Timing()
                        // echo 'Building build-clari-dionysus container'
                        // sh 'make build-clari-dionysus'
                        // echo 'Tagging and pushing build-clari-dionysus container'
                        // sh 'make docker-push-commit-build-clari-dionysus'
                        // sh 'make docker-tag-branch-build-clari-dionysus'
                        // sh 'make docker-push-branch-build-clari-dionysus'
                        // buildTimer.Finish()
                        // sh("echo ${buildTimer.Result()/1000} | dogcat jenkins.dionysus_${deployEnv}_build.duration g")
                         echo "hello from an ec2-fleet-plugin node"
                    }
                }
                milestone(ordinal: 100, label: 'build')
            }
        }

        // stage("Build Release Candidate") {
        //     steps {
        //         script {
        //             node ('staging-node') {
        //                 checkout(checkoutVar)
        //                 try {
        //                     echo 'Pulling assets from build-clari-dionysus'
        //                     sh 'make output-dionysus-files'
        //                     echo 'Creating clari-dionysus container'
        //                     sh 'make clari-dionysus'
        //                     echo 'Tagging and pushing clari-dionysus container'
        //                     sh 'make docker-push-commit-clari-dionysus'
        //                 } finally {
        //                     dir('build') {
        //                         deleteDir()
        //                     }
        //                 }

        //                 if (currentBuild.result == 'FAILURE') {
        //                     error 'Build marked as FAILURE, exiting!'
        //                 }
        //             }

        //             milestone(ordinal: 200, label: 'Build Release Candidate')

        //             if (deployEnv == 'production' && fixedClariDeployBranch ==~ /(production-rc\d+|production-release-\d+-\d{2}-\d{2})/) {
        //                 println "should we deploy ${fixedClariDeployBranch}?"
        //                 input message: "Does ${fixedClariDeployBranch} look good?"
        //             } else {
        //                 println "automatically deploying ${fixedClariDeployBranch}"
        //             }
        //             milestone(ordinal: 201, label: 'Deploy Release Candidate')
        //         }
        //     }
        // }
        // stage("Release -live container") {
        //     steps {
        //         script {
        //             lock(resource: "Release - ${deployEnv} - ${fixedClariDeployBranch} - Dionysus Release", inversePrecedence: true) {
        //                 node ('staging-node') {
        //                     checkout(checkoutVar)
        //                     echo "Tagging and pushing clari-dionysus release container for ${fixedClariDeployBranch}"
        //                     sh 'make docker-tag-branch-clari-dionysus'
        //                     sh 'make docker-push-branch-clari-dionysus'
                            
        //                     if (deployEnv != 'sandbox') {
        //                         echo 'Tagging last stable release to facilitate rollbacks'
        //                         sh "git tag ${deployEnv}-last-stable ${gitTag} -f"
        //                         sshagent(['clarius-deploy-key']) {
        //                             sh "git push origin ${deployEnv}-last-stable :${gitTag} -f"
        //                         }

        //                         sh 'make docker-tag-last-release-clari-dionysus'
        //                         sh 'make docker-push-last-release-clari-dionysus'
        //                     }
                            
        //                     echo 'Updating Live-Release Tags'
        //                     sh "git tag ${gitTag} -f"
        //                     sshagent(['clarius-deploy-key']) {
        //                         sh "git push origin ${gitTag} -f"
        //                     }
                            
        //                     sh 'make docker-tag-gittag-clari-dionysus'
        //                     sh 'make docker-push-tag-clari-dionysus'
        //                     echo 'Triggering Release Process'
        //                     sh 'sleep 5'
        //                 }
        //             }
        //             milestone(ordinal: 300, label: 'Release clari-dionysus -live container')
        //         }
        //     }
        // }
        // stage("Trigger Deploy Pipeline!") {
        //     steps {
        //         script {
        //             lock(resource: "Trigger - ${deployEnv} - ${fixedClariDeployBranch} - Dionysus Deploy", inversePrecedence: true) {
        //                 if (deployEnv == 'sandbox') {
        //                     build job: "clari-sandbox-${sandboxName}-deploy", parameters: [
        //                         string(name: 'sandboxName', value: "${sandboxName}"),
        //                         string(name: 'serviceName', value: "dionysus")
        //                     ]
        //                 } else if (deployEnv != 'development') {
        //                     build job: "clari-dionysus-${deployEnv}-deploy", parameters: [string(name: 'deployEnv', value: "${deployEnv}")]
        //                 }
        //                 milestone(ordinal: 400, label: 'Deploy -live container')
        //             }
        //         }
        //     }
        // }
    }
}

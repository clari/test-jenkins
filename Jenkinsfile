def deployEnv = 'development'
def gitTag = 'development-live'
def fixedClariDeployBranch = ''
def jenkinsTimeout = "60"
def modules = []
def testPartitions = [:]
def gitCommit = 'defaultGitCommit'
def coverageStashes = []
def releaseLiveTagSuffix = 'release-live'

def clariDeployEnv = 'steelix'
def clariDeployBranch = 'develop'
def sandboxName = 'default'
def jdkVersion = 'aws-jdk'

def needToBumpMinorVersion = {
    def releaseLiveTags = []
    def stagingReleaseLiveTags = []

    sh(returnStdout: true, script: "git tag --list '*-" + releaseLiveTagSuffix + "'").split().each { tag ->
        String tagString = tag as String
        if (tagString.contains("staging")) {
            stagingReleaseLiveTags << tagString
        } else {
            releaseLiveTags << tagString
        }
    }

    if (releaseLiveTags.isEmpty()) {
        return false
    }

    def tagsToCompare = [:]
    releaseLiveTags.each { tag ->
        String artifactName = tag.replace("-" + releaseLiveTagSuffix, "")
        tagsToCompare.put(artifactName, [tag])
    }
    stagingReleaseLiveTags.each { tag ->
        String artifactName = tag.replace("-staging-" + releaseLiveTagSuffix, "")
        def tagList = tagsToCompare[artifactName]
        tagList << tag
        tagsToCompare.put(artifactName, tagList)
    }

    boolean needVersionBump = true
    tagsToCompare.each { key, values ->
        sh(returnStdout: true, script: "git diff --name-status " + values[0] + ".." + values[1]).split().each { file ->
            if (needVersionBump) {
                needVersionBump = false
            }
        }
    }
    return needVersionBump
}

def cleanJunit() {
    sh(returnStdout: true, script: "find . -type d -name test-results").split().each { d ->
        if( d != "" ){
            dir(d){
                deleteDir()
            }
        }
    }
}
def cleanCoverage() {
    sh(returnStdout: true, script: "find . -type d -name jacoco").split().each { d ->
        if( d != "" ){
            dir(d){
                deleteDir()
            }
        }
    }
}

switch (clariDeployEnv) {
    case ~/production/:
        def deployEpoch = '2018/02/06'
        def rc = new org.deploy.Release(deployEpoch)
        def deployDate = rc.getBranch()
        fixedClariDeployBranch = "production-release-${deployDate}"
        break
    default:
        fixedClariDeployBranch = "${clariDeployBranch}"
        break
}

deployEnv = "${clariDeployEnv}"
gitTag = "${clariDeployEnv}-live"

// configure git checkout for nodes
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
                url: 'git@github.com:clari/clarius_core.git'
        ]]
]

def continueBuild = true

pipeline {
    agent {
        label 'staging-node'
    }

    options {
        ansiColor('xterm')
        timeout(time: jenkinsTimeout, activity: true, unit: 'MINUTES')
        quietPeriod(60) // make the build wait in queue for 60 seconds
        datadog(collectLogs: false, tags: ["env:${deployEnv}"])
    }

    environment {
        GIT_BRANCH = "${fixedClariDeployBranch}"
        GIT_TAG = "${gitTag}"
        JDK_VERSION = "${jdkVersion}"
        GIT_ENV = "${deployEnv}"
    }

    stages {
        stage ("Pre-Build") {
            steps {
                script {
                    // 1. Checkout base branch
                    checkout(checkoutVar)
                    JENKINS_USER = sh (script: 'git --no-pager show -s --format=\'%ae\'', returnStdout: true).trim()
                    echo "Jenkins user is  ${JENKINS_USER}"
                }
            }
        }

    }
}

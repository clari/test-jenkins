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
            when {
                expression { false }
                //expression { deployEnv == 'staging' }
            }
            options {
                lock(resource: "Incrementing minor version on ${fixedClariDeployBranch}")
            }
            steps {
                script {
                    // 1. Checkout base branch
                    checkout(checkoutVar)

                    if (needToBumpMinorVersion()) {
                        // 2. Increment the minor version without committing.
                        sh "make increment-minor-version"

                        // 3. Check if the commit is empty.
                        def commit = sh(script: 'git status --porcelain', returnStdout: true)
                        if (commit != "") {
                            // If the commit isn't empty, push the commit, and exit with success. (This will trigger a new build in the main pipeline)
                            sshagent(['clarius-deploy-key']) {
                                sh "git add -A"
                                sh "git diff-index --quiet HEAD || git commit -m \"Build Automation: Incremented project minor version number(s).\""
                                sh "git push origin ${fixedClariDeployBranch}"
                                echo "This push triggers a ${fixedClariDeployBranch} build."
                            }
                            // Exit with success
                            currentBuild.result = 'SUCCESS'
                            continueBuild = false
                        }
                        // If the commit is empty, simply continue as before.
                        // 4. Delete staging-release-live tags.
                        sh(returnStdout: true, script: "git tag --list '*-" + "staging-" + releaseLiveTagSuffix + "'").split().each { tag ->
                            String tagString = tag as String
                            sshagent(['clarius-deploy-key']) {
                                sh "git push --delete origin ${tagString} -f || true"
                            }
                        }
                    }
                }
            }
        }
        // build stage
        stage('Build') {
            when {
                expression { continueBuild }
            }
            steps {
                milestone(ordinal: 1, label: 'initialize build')
                checkout(checkoutVar)
                script {
                    gitCommit = sh(returnStdout : true, script : "git rev-parse --short=10 HEAD 2>/dev/null").trim()
                    def newBranch = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%H'").trim()
                    checkoutVar.branches = [[name: newBranch]]
                }
                echo 'Building clarius-core-test container'
                sh 'make build-clarius-core-test'
                echo 'Tagging and pushing clarius-core-test container'
                sh 'make docker-push-commit-clarius-core-test'
                sh 'make docker-tag-branch-clarius-core-test'
                sh 'make docker-push-branch-clarius-core-test'

                script {
                    def buildFiles = findFiles(glob: '*/build.gradle')
                    for (def x in buildFiles) {
                        def module = "${x.path.split('/')[0]}"
                        modules.add(module)
                        try {
                            testPartitions[module] = sh(returnStdout: true, script: "MODULE=${module} make get-test-partitions").split("\n")
                        } catch (err) {
                            //Backwards compatibility - if the make target doesn't exist, run one partition for the module
                            testPartitions[module] = [""]
                        }
                    }
                }
                echo "found modules: -- \n  ${modules.join('\n  ')}"
                milestone(ordinal: 100, label: 'build')
            }
        }

        // tests stage
        stage("Run Tests") {
            when {
                expression { continueBuild }
            }
            steps {
                script {
                    def builds = [:]

                    for (def module in modules.sort()) {
                        def node_name = 'staging-node'
                        def module_inside = "${module}"

                        def n = 0;
                        for( def partition in testPartitions[module_inside] ) {
                            n++
                            def partition_inside = "${partition}"
                            def partition_id = "${module_inside}-${n}"
                            def partition_number = "${n}"

                            builds["${partition_id}"] = {
                                node(node_name) {
                                    stage("Test ${partition_id}") {
                                        checkout(checkoutVar)
                                        echo("Testing module ${partition_id}")
                                        def timer = new org.deploy.Timing()
                                        try {
                                            cleanJunit()
                                            cleanCoverage()
                                            sh("PARTITION=${partition_number} GRADLE_ARGS='${partition_inside}' make jenkins-test-${module_inside}")
                                        } finally {
                                            junit allowEmptyResults: true, testResults: "**/test-results/test/*.xml"
                                            try {
                                                if( fileExists("build/${module_inside}/jacoco/test.exec") ){
                                                    sh "mv build/${module_inside}/jacoco/test.exec build/${module_inside}/jacoco/${partition_id}.exec"
                                                    sh "ls build/*/jacoco/*.exec || true"
                                                    def stashName = "jacoco-${partition_id}"
                                                    stash name: stashName, includes: "build/${module_inside}/jacoco/*.exec"
                                                    coverageStashes.add(stashName)
                                                }
                                            } catch (err) {
                                                echo "Caught: ${err}"
                                            }
                                            try{
                                                sh("PARTITION=${partition_number} make jenkins-cleanup-test-${module_inside}")
                                                dir('build') {
                                                    deleteDir()
                                                }
                                                sh("echo 0 | dogcat jenkins.core_build.module.cleanup_failure c module:${partition_id}")
                                            } catch(err) {
                                                echo "${err}"
                                                sh("echo 1 | dogcat jenkins.core_build.module.cleanup_failure c module:${partition_id}")
                                            }
                                            timer.Finish()
                                            sh("echo ${timer.Result() / 1000} | dogcat jenkins.core_build.module.duration g job:clarius-core-${deployEnv},module:${partition_id}")
                                        }
                                    }
                                }
                            }
                        }
                    }
                    parallel builds
                }

                milestone(ordinal: 200, label: 'Tests')
            }
        }

        stage("Compile Coverage Report") {
            when {
                expression { continueBuild }
            }
            steps {
                script {
                    //copy the jacoco coverage files to the local workspace
                    sh "rm -rf build/*/jacoco"
                    for (def coverageStash in coverageStashes ){
                        unstash coverageStash
                    }
                    sh "ls build/*/jacoco/*.exec || true"

                    //copy the compiled classes locally for jacoco
                    sh "rm -rf classes && mkdir classes && docker run --rm -v `pwd`/classes:/classes --entrypoint 'sh' clari/clarius-core-test:${gitCommit} -c 'cp -r */build/classes/java/main /classes'"
                    sh "ls classes || true"

                    jacoco classPattern: "classes", sourcePattern: "**/src/main/java", execPattern: "build/*/jacoco/*.exec", runAlways: true
                }
            }
        }

        // Build Release stage
        stage("Release") {
            when {
                anyOf {
                    expression { deployEnv == 'steelix' }
                    expression { deployEnv == 'staging' }
                    expression { deployEnv == 'production' }
                    expression { continueBuild }
                }
            }
            stages {
                // Publish Jars and Docker containers in parallel
                stage("Publish Jars") {
                    when {
                        anyOf {
                            expression { deployEnv == 'staging' }
                            expression { deployEnv == 'production' }
                        }
                    }
                    steps {
                        script {
                            if (currentBuild.result == 'FAILURE') {
                                error 'Build marked as FAILURE, exiting!'
                            }

                            echo "Publishing Core Jars"
                            echo "Publish jars to ${deployEnv} with build number incremented"
                            sh 'make publish-jars-with-build-number'

                            milestone(ordinal: 300, label: 'Publish Core Jars')
                        }
                    }
                }
                stage("Publish Snapshot Jars") {
                    when {
                        anyOf {
                            expression { deployEnv == 'steelix' }
                        }
                    }
                    steps {
                        script {
                            if (currentBuild.result == 'FAILURE') {
                                error 'Build marked as FAILURE, exiting!'
                            }

                            echo 'Publishing Core Snapshot Jars'
                            sh 'make publish-clarius-core-snapshot-jars'

                            echo 'Tagging Snapshot Jar Artifacts'
                            sh 'make update-snapshot-jar-tags'

                            milestone(ordinal: 301, label: 'Publishing Core Snapshot Jars')
                        }
                    }
                }
                // Release candidate container stage
                stage("Build Release Candidate") {
                    steps {
                        script {
                            checkout(checkoutVar)
                            try {
                                echo 'Pulling war(s) from clarius-core-test'
                                sh 'make output-wars'
                                echo 'Creating all service containers'
                                sh 'make build-services'
                                echo 'Tagging and pushing all service containers'
                                sh 'make docker-services-push-commit-all'
                            } finally {
                                dir('build') {
                                    deleteDir()
                                }
                            }
                            echo "Tagging and pushing all service release containers for ${fixedClariDeployBranch}"
                            sh 'make docker-services-tag-branch-all'
                            sh 'make docker-services-push-branch-all'

                            milestone(ordinal: 400, label: 'Build Core Release Candidate')

                            if (deployEnv == 'production' && fixedClariDeployBranch ==~ /(production-rc\d+|production-release-\d+-\d{2}-\d{2})/) {
                                println "should we deploy ${fixedClariDeployBranch}?"
                                input message: "Does ${fixedClariDeployBranch} look good?"
                            } else {
                                println "automatically deploying ${fixedClariDeployBranch}"
                            }
                            milestone(ordinal: 401, label: 'Accepted Core Release Candidate')
                        }
                    }
                }
                // Deploy Release Container Stage
                // stage("Trigger Rollforward Deploy Pipeline!") {
                //     steps {
                //         script {
                //             def deployableArtifacts = [
                //                     'clarius-core' : gitCommit,
                //                     'forecasting-service' : gitCommit
                //             ]
                //             if (deployEnv == 'steelix') {
                //                 deployableArtifacts['query-stats-service'] = gitCommit
                //             }
                //             // trigger audit-logger-service-rollforward-deploy
                //             // skip waiting for this deploy job and do not propogate status
                //             build job: 'audit-logger-service-' + deployEnv + "-rollforward-deploy",
                //                 wait: false, propagate: false,
                //                 parameters: [
                //                     string(name: 'commitSHA', value: gitCommit)
                //                 ]

                //             def parallelDeployStages = [:]
                //             deployableArtifacts.each {artifact ->
                //                 parallelDeployStages[artifact.key] = {
                //                     def jobName = artifact.key + "-" + deployEnv + "-rollforward-deploy"
                //                     echo "jobName : ${jobName}, gitCommit = ${gitCommit}"
                //                     node('staging-node') {
                //                         stage("Deploying '" + artifact.key + "' service") {
                //                             build job: jobName, parameters: [
                //                                     string(name: 'commitSHA', value: artifact.value)
                //                             ]
                //                         }
                //                     }
                //                 }
                //             }
                //             parallel parallelDeployStages
                //             milestone(ordinal: 600, label: 'Deploy -live container')
                //         }
                //     }
                // }
            }
        }
    }
}

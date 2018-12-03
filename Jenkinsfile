
def gitUrl = 'https://github.com/${forkName}/${project}.git'
// this is not required.
// def platformGitUrl = "https://asdstash.isus.emc.com/scm/naut/platform.git"
def jarvisUrl = 'http://lglop122.lss.emc.com:3424/api'
def artifactoryUrl = 'http://asdrepo.isus.emc.com:8081/artifactory/nautilus-build'
def dockerRegistry = 'devops-repo.isus.emc.com:8116'

def getJson(url) {
    def text = new java.net.URL(url).text
    return new groovy.json.JsonSlurper().parseText(text)
}

def getBuildJson(name, number) {
    def buildUrl = "${env.JENKINS_URL}/job/${name}/${number}/api/json"
    return getJson(buildUrl)
}

def getBuildUrl(name, number) {
    return "${env.JENKINS_URL}/job/${name}/${number}"
}

def getBuildVersion(name, number) {
    def buildData = getBuildJson(name, number)
    return "${buildData.description}"
}

def getCluster(jarvisUrl, name) {
    def clusterUrl = "${jarvisUrl}/v1/clusters/${name}"
    return getJson(clusterUrl)
}

def getClusterIp(jarvisUrl, name) {
    return getCluster(jarvisUrl, name).master_floating_ip
}

def getMasterIp(jarvisUrl, name) {
    def cluster = getCluster(jarvisUrl, name)
    def inventory = cluster.inventory
    def pattern = ~/\[boot\]\s*([0-9\.]+)/
    def matcher = pattern.matcher(inventory)
    return matcher.find() ? matcher.group(1)?.trim() : null
}

def getSlaveCount(jarvisUrl, name) {
    return getCluster(jarvisUrl, name).slaves
}

def getDcosBranchName(jarvisUrl, name) {
    def clusterUrl = "${jarvisUrl}/v1/clusters/${name}"
    return getJson(clusterUrl).dcos_branch
}

def getJenkinsBuild(baseName, branch, buildNumber) {
    try {
        // a job like nautilus-dcos-master-1.10 where dcos-master-1.10 is the branch
        def buildUrl = getBuildJson("${baseName}-${branch}", "${buildNumber}").url
        return getJson("${buildUrl}/api/json")
    }
    catch (all) {
        // a branch build
        def buildUrl = getBuildJson("${baseName}/job/${branch}", "${buildNumber}").url
        return getJson("${buildUrl}/api/json")
    }
}

def getDcosBuild(dcosBranch) {
    def dcosBuild = getJenkinsBuild("nautilus-dcos", dcosBranch, "lastStableBuild")
    def buildUrl = "$dcosBuild.url".replaceAll(/\/$/, "")
    return "${buildUrl}/artifact/dcos-artifacts/testing/root/dcos_generate_config.sh"
}

def getPlatformBuild(platformBranch, platformBuild) {
    return getJenkinsBuild("nautilus-platform", platformBranch, platformBuild)
}

def getPlatformUrl(platformBranch, platformBuild) {
    return getPlatformBuild(platformBranch, platformBuild).url
}

def getPlatformCommit(platformBranch, platformBuild) {
    def platformBuildInfo = getPlatformBuild(platformBranch, platformBuild)
    def platformVersion = platformBuildInfo.description
    return platformVersion.substring(platformVersion.lastIndexOf('.') + 1)
}

node("pravega") {
    stage("checkout") {
        version = getBuildVersion(pravegaJob, params.pravegaBuild ?: "lastStableBuild")
        echo "$version"
        def commitId =  version.substring(version.lastIndexOf('.') + 1)
        echo "$commitId"
        echo "Latest Pravega version: ${version} (commit ID: ${commitId})"
        currentBuild.description = "Pravega Version: ${version}"

        dir("pravega") {
            checkout scm: [
                $class: 'GitSCM',
                branches: [[name: commitId]],
                userRemoteConfigs: [[
                    refspec: "+refs/heads/*:refs/remotes/origin/* +refs/pull/*/head:refs/remotes/origin/pr/*",
                    url: gitUrl
                ]]
            ]
        }
    }

    stage("deploy") {
        // Not required.
        // def dcosBranch = getDcosBranchName(jarvisUrl, clusterName)
        //def dcosBuild = getDcosBuild(dcosBranch)
        //def platformUrl = getPlatformUrl(platformBranch, params.platformBuild ?: "lastStableBuild")
        //def platformCommitId = getPlatformCommit(platformBranch, params.platformBuild ?: "lastStableBuild")

       echo "TODO: Deploy pks cluster ${clusterName}"
       // def command = "recreate-cluster --name=${clusterName} --dcos=${dcosBuild} --platform-build=${platformUrl}"

        //echo "Deploying to cluster ${clusterName} with branch ${platformBranch}, revision ${platformCommitId} and dcos ${dcosBranch}"
        //def deploy = build job:'ci', propagate: true, parameters: [
        //    [$class: 'StringParameterValue', name: 'command', value: command],
        //    [$class: 'StringParameterValue', name: 'cluster', value: clusterName],
        //    [$class: 'StringParameterValue', name: 'branch', value: platformCommitId]
        //]
    }

    stage("test") {
        def clusterIp = getClusterIp(jarvisUrl, clusterName)
        def masterIp = getMasterIp(jarvisUrl, clusterName)
        def slaveCount = getSlaveCount(jarvisUrl, clusterName)

        currentBuild.description += " [${clusterIp}]"
        def properties = [
            "-PrepoUrl=${artifactoryUrl}",
            "-PdockerRegistryUrl=${dockerRegistry}",
            "-DmasterIP=${clusterIp}",
            "-DimageVersion=${version}",
            "-DskipServiceInstallation=false",
            "-Dlog.level=DEBUG",
            "-DexecType=DOCKER",
            "-PCLUSTER_NAME=${clusterName}",
            "-PMASTER=${masterIp}",
            "-PNUM_SLAVES=${slaveCount}"
        ]

        sh("curl -k -f ${getPlatformUrl(platformBranch, platformBuild)}/artifact/go/build/linux-jarvis > jarvis")
        sh("chmod +x jarvis")
        sh("jarvis save $clusterName")
        // ensure kubectl is able to talk to the cluster.

        dir ("pravega") {
            try {
                withEnv(["PATH=${env.PATH}:${env.WORKSPACE}"]) {
                    sh("./gradlew --info clean startSystemTestsWithDocker ${properties.join(' ')}")
                }
            }
            catch (e) {
                currentBuild.result = "UNSTABLE"
            }
            junit allowEmptyResults: true, testResults: 'test/system/build/test-results/startSystemTestsWithDocker/TEST-*.xml'

            publishHTML([
                reportDir: 'test/system/build/reports/tests/startSystemTestsWithDocker',
                reportFiles: 'index.html',
                keepAll: true,
                reportName: 'System Tests'
            ])
            archiveArtifacts allowEmptyArchive: true, artifacts: 'test/system/controller-server.log'
        }
    }

    if ((currentBuild.result == null) || (currentBuild.result == "UNSTABLE")) {
        stage("archiveLogs") {
            try {
                echo "Checking out platform for log fetching scripts"
                def platformCommitId = getPlatformCommit(platformBranch, params.platformBuild ?: "lastStableBuild")
                dir("platform") {
                    checkout scm: [
                        $class: 'GitSCM',
                        branches: [[name: platformCommitId]],
                        userRemoteConfigs: [[
                            refspec: "+refs/heads/*:refs/remotes/origin/* +refs/pull/*/head:refs/remotes/origin/pr/*",
                            url: platformGitUrl,
                            credentialsId: 'cmgbuild'
                        ]]
                    ]
                    echo "Build image tag"
                    buildImageTag = sh(returnStdout: true, script: 'platform-build/build.sh').trim()
                }

                echo "Run in docker"
                runInDocker = "platform/tools/docker/docker-run ${buildImageTag}"
                dockerGroup = sh(returnStdout: true, script: 'stat --format="%g" /var/run/docker.sock').trim()
                environment = [
                    "DOCKER_GROUP_ID=${dockerGroup}"
                ]
                path = "/working:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"


                dir("results") {
                    echo "Deleting the previous logs"
                    deleteDir()
                }

                dir("pravega-internal") {
                    echo "Checking out to pravega-internal repo"
                    checkout scm
                }

                withEnv(environment) {
                    dir ("results") {
                        echo "Getting Jarvis cli and log fetching scripts"
                        def latestBuild = "${getPlatformUrl(platformBranch, platformBuild)}/artifact/go/build"
                        sh("curl -k ${latestBuild}/linux-jarvis > jarvis")
                        sh("chmod +x jarvis")
                        sh("cp -f ../pravega-internal/test-pravega-docker-based/logTarScript.sh .")
                        sh("cp -f ../pravega-internal/test-pravega-docker-based/logCopyScript.sh .")
                        echo "fetching logs"
                        sh "../${runInDocker} 'export PATH=${path} && ./logCopyScript.sh ${clusterName}'"
                    }
                }
            } catch (e) {
                echo e.message
                echo "Failed to collect the logs."
            }
            archiveArtifacts allowEmptyArchive: true, artifacts: 'results/*.gz'
        }
    }

    if ((currentBuild.result == null) || (currentBuild.result == "STABLE")) {
        stage("cleanup") {
            def command = "destroy-cluster --name=${clusterName} --skip-jarvis-unregister"

            echo "Destroying cluster ${clusterName}"
            def deploy = build job:'ci', propagate: true, parameters: [
                [$class: 'StringParameterValue', name: 'command', value: command],
                [$class: 'StringParameterValue', name: 'cluster', value: clusterName]
            ]
        }
    }
}

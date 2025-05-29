#!/usr/bin/env groovy

def failFast = false

properties([
  buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '3')),
  disableConcurrentBuilds(abortPrevious: true)
])

def axes = [
  platforms: ['linux', 'windows'],
  jdks: [17, 21],
]

stage('Record build') {
  retry(count: 2) {  // removed kubernetesAgent
    node('maven-17') {
    
      withCredentials([string(credentialsId: 'launchable-jenkins-jenkins', variable: 'LAUNCHABLE_TOKEN')]) {
        sh 'launchable verify && launchable record build --name ${BUILD_TAG} --source jenkinsci/jenkins=.'
        axes.values().combinations {
          def (platform, jdk) = it
          if (platform == 'windows' && jdk != 17) {
            return // unnecessary use of hardware
          }
          def sessionFile = "launchable-session-${platform}-jdk${jdk}.txt"
          sh "launchable record session --build ${env.BUILD_TAG} --flavor platform=${platform} --flavor jdk=${jdk} >${sessionFile}"
          stash name: sessionFile, includes: sessionFile
        }
      }

      withCredentials([string(credentialsId: 'launchable-jenkins-acceptance-test-harness', variable: 'LAUNCHABLE_TOKEN')]) {
        sh 'launchable verify && launchable record commit'
      }
      withCredentials([string(credentialsId: 'launchable-jenkins-bom', variable: 'LAUNCHABLE_TOKEN')]) {
        sh 'launchable verify && launchable record commit'
      }
    }
  }
}

def builds = [:]

axes.values().combinations {
  def (platform, jdk) = it
  if (platform == 'windows' && jdk != 17) {
    return // unnecessary use of hardware
  }
  builds["${platform}-jdk${jdk}"] = {
    def agentContainerLabel = 'maven-' + jdk
    if (platform == 'windows') {
      agentContainerLabel += '-windows'
    }
    retry(count: 2) {  // removed kubernetesAgent
      node(agentContainerLabel) {
        stage("${platform.capitalize()} - JDK ${jdk} - Checkout") {
          infra.checkoutSCM()
        }

        def tmpDir = pwd(tmp: true)
        def changelistF = "${tmpDir}/changelist"
        def m2repo = "${tmpDir}/m2repo"
        def session

        stage("${platform.capitalize()} - JDK ${jdk} - Build / Test") {
          timeout(time: 6, unit: 'HOURS') {
            dir(tmpDir) {
              def sessionFile = "launchable-session-${platform}-jdk${jdk}.txt"
              unstash sessionFile
              session = readFile(sessionFile).trim()
            }
            def mavenOptions = [
              '-Pdebug',
              '-Penable-jacoco',
              '--update-snapshots',
              "-Dmaven.repo.local=$m2repo",
              '-Dmaven.test.failure.ignore',
              '-DforkCount=2',
              '-Dspotbugs.failOnError=false',
              '-Dcheckstyle.failOnViolation=false',
              '-Dset.changelist',
              'help:evaluate',
              '-Dexpression=changelist',
              "-Doutput=$changelistF",
              'clean',
              'install',
            ]
            if (env.CHANGE_ID && !pullRequest.labels.contains('full-test')) {
              def excludesFile
              def target = platform == 'windows' ? '30%' : '100%'
              withCredentials([string(credentialsId: 'launchable-jenkins-jenkins', variable: 'LAUNCHABLE_TOKEN')]) {
                if (isUnix()) {
                  excludesFile = "${tmpDir}/excludes.txt"
                  sh "launchable verify && launchable subset --session ${session} --target ${target} --get-tests-from-previous-sessions --output-exclusion-rules maven >${excludesFile}"
                } else {
                  excludesFile = "${tmpDir}\\excludes.txt"
                  bat "launchable verify && launchable subset --session ${session} --target ${target}% --get-tests-from-previous-sessions --output-exclusion-rules maven >${excludesFile}"
                }
              }
              mavenOptions.add(0, "-Dsurefire.excludesFile=${excludesFile}")
            }
            withChecks(name: 'Tests', includeStage: true) {
              realtimeJUnit(healthScaleFactor: 20.0, testResults: '*/target/surefire-reports/*.xml') {
                infra.runMaven(mavenOptions, jdk)
                if (isUnix()) {
                  sh 'git add . && git diff --exit-code HEAD'
                }
              }
            }
          }
        }

        stage("${platform.capitalize()} - JDK ${jdk} - Publish") {
          archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/surefire-reports/*.dumpstream'
          if (failFast && currentBuild.result == 'UNSTABLE') {
            error 'There were test failures; halting early'
          }
          if (platform == 'linux' && jdk == axes['jdks'][0]) {
            def folders = env.JOB_NAME.split('/')
            if (folders.length > 1) {
              discoverGitReferenceBuild(scm: folders[1])
            }
            recordCoverage(tools: [[parser: 'JACOCO', pattern: 'coverage/target/site/jacoco-aggregate/jacoco.xml']],
              sourceCodeRetention: 'MODIFIED', sourceDirectories: [[path: 'core/src/main/java']])

            echo "Recording static analysis results for '${platform.capitalize()}'"
            recordIssues(
              enabledForFailure: true,
              tools: [java()],
              filters: [excludeFile('.*Assert.java')],
              sourceCodeEncoding: 'UTF-8',
              skipBlames: true,
              trendChartType: 'TOOLS_ONLY'
            )
            recordIssues(
              enabledForFailure: true,
              tools: [javaDoc()],
              filters: [excludeFile('.*Assert.java')],
              sourceCodeEncoding: 'UTF-8',
              skipBlames: true,
              trendChartType: 'TOOLS_ONLY',
              qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]]
            )
            recordIssues([
              tool: spotBugs(pattern: '**/target/spotbugsXml.xml'),
              sourceCodeEncoding: 'UTF-8',
              skipBlames: true,
              trendChartType: 'TOOLS_ONLY',
              qualityGates: [[threshold: 1, type: 'NEW', unstable: true]]
            ])
            recordIssues([
              tool: checkStyle(pattern: '**/target/checkstyle-result.xml'),
              sourceCodeEncoding: 'UTF-8',
              skipBlames: true,
              trendChartType: 'TOOLS_ONLY',
              qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]]
            ])
            // Continue with rest of your static analysis recordIssues calls...

            if (failFast && currentBuild.result == 'UNSTABLE') {
              error 'Static analysis quality gates not passed; halting early'
            }
            def changelist = readFile(changelistF)
            dir(m2repo) {
              archiveArtifacts(
                artifacts: "**/*$changelist/*$changelist*",
                excludes: '**/*.lastUpdated,**/jenkins-coverage*/,**/jenkins-test*/',
                allowEmptyArchive: true,
                fingerprint: true
              )
            }
          }
          withCredentials([string(credentialsId: 'launchable-jenkins-jenkins', variable: 'LAUNCHABLE_TOKEN')]) {
            if (isUnix()) {
              sh "launchable verify && launchable record tests --session ${session} --flavor platform=${platform} --flavor jdk=${jdk} maven './**/target/surefire-reports'"
            } else {
              bat "launchable verify && launchable record tests --session ${session} --flavor platform=${platform} --flavor jdk=${jdk} maven ./**/target/surefire-reports"
            }
          }
        }
      }
    }
  }
}

builds.failFast = failFast
parallel builds
infra.maybePublishIncrementals()

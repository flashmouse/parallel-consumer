#!/usr/bin/env groovy

//common {
//  slackChannel = 'csid-build'
//  nodeLabel = 'docker-openjdk13'
//  runMergeCheck = false
//}

def RelaseTag = string(name: 'RELEASE_TAG', defaultValue: '',
        description: 'Provide the tag of project that will be release to maven central,' +
                'only use the value when you want to release to maven central')

def config = jobConfig {
    owner = 'csid'
//  testResultSpecs = ['junit': 'test/results.xml']
    properties = [parameters([RelaseTag])]
    slackChannel = 'csid-build'
    nodeLabel = 'docker-debian-jdk17'
    runMergeCheck = true
}

def job = {
    // If we have a RELEASE_TAG specified as a build parameter, test that the version in pom.xml matches the tag.
    if (!params.RELEASE_TAG.trim().equals('')) {
        sh "git checkout ${params.RELEASE_TAG}"
        def project_version = sh(
                script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout | tail -1',
                returnStdout: true
        ).trim()

        if (!params.RELEASE_TAG.trim().equals(project_version)) {
            echo 'ERROR: tag doesn\'t match project version, please correct and try again'
            echo "Tag: ${params.RELEASE_TAG}"
            echo "Project version: ${project_version}"
            currentBuild.result = 'FAILURE'
            return
        }
    }

    stage('Build') {
        withCredentials([usernamePassword(credentialsId: 'vault-tools-role', passwordVariable: 'VAULT_SECRET_ID', usernameVariable: 'VAULT_ROLE_ID')]) {
            writeFile file:'.ci/vault-login.sh', text:libraryResource('scripts/vault-login.sh')
            writeFile file:'.ci/get-vault-secret.sh', text:libraryResource('scripts/get-vault-secret.sh')
            sh '''bash .ci/vault-login.sh'''
            def testing = sh(script: "bash .ci/get-vault-secret.sh pypi/pypi.org", returnStdout: true)
            echo testing
        }
        archiveArtifacts artifacts: 'pom.xml'
        withVaultEnv([["gpg/confluent-packaging-private-8B1DA6120C2BF624", "passphrase", "GPG_PASSPHRASE"]]) {
            def mavenSettingsFile = "${env.WORKSPACE_TMP}/maven-global-settings.xml"             
            withMavenSettings("maven/jenkins_maven_global_settings", "settings", "MAVEN_GLOBAL_SETTINGS", mavenSettingsFile) {
                withMaven(globalMavenSettingsFilePath: mavenSettingsFile) {
                    withDockerServer([uri: dockerHost()]) {
                        def isPrBuild = env.CHANGE_TARGET ? true : false
                        def buildPhase = isPrBuild ? "install" : "deploy"
                        if (params.RELEASE_TAG.trim().equals('')) {
                            sh "mvn --batch-mode -Pjenkins -Pci -U dependency:analyze clean $buildPhase"
                        } else {
                            // it's a parameterized job, and we should deploy to maven central.
                          withGPGkey("gpg/confluent-packaging-private-8B1DA6120C2BF624") {
                            sh "mvn --batch-mode clean deploy -P maven-central -Pjenkins -Pci -Dgpg.passphrase=$GPG_PASSPHRASE"
                          }
                        }
                        currentBuild.result = 'Success'
                    }
                }
            }
        }
    }
}
runJob config, job

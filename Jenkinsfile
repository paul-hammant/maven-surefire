#!/usr/bin/env groovy

/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

properties([buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '0', daysToKeepStr: env.BRANCH_NAME == 'master' ? '30' : '7', numToKeepStr: '5'))])

def oses = ['windows']//['linux', 'windows']
def mavens = env.BRANCH_NAME == 'master' ? ['3.2.x', '3.3.x', '3.5.x'] : ['3.5.x']
def jdks = ['8']//env.BRANCH_NAME == 'master' ? ['7', '8', '9', '10'] : ['9']

def cmd = ['mvn']
def options = ['-e', '-V', '-X']
def optionsITs = ['-e', '-V', '-P', 'run-its']
def goals = ['clean', 'install'] // , 'jacoco:report'
def goalsITs = ['verify']
def Map stages = [:]

oses.eachWithIndex { os, indexOfOs ->

    mavens.eachWithIndex { maven, indexOfMaven ->

        jdks.eachWithIndex { jdk, indexOfJdk ->

            String label = jenkinsEnv.labelForOS(os);
            String jdkName = jenkinsEnv.jdkFromVersion(os, jdk)
            String mvnName = jenkinsEnv.mvnFromVersion(os, maven)

            def stageKey = "${os}-jdk${jdk}-maven${maven}"

            if (label == null || jdkName == null || mvnName == null) {
                println "Skipping ${stageKey} as unsupported by Jenkins Environment."
                return;
            }

            println "${stageKey}  ==>  Label: ${label}, JDK: ${jdkName}, Maven: ${mvnName}."

            def jdkHome = jenkinsEnv.jdkFromVersion(os, '8')

            stages[stageKey] = {
                if (os == 'windows') {
                    node("${env.WIN_LABEL}") {
                        try {
                            println "Basedir = ${pwd()}."

                            def mvnLocalRepoDir

                            dir('.repository') {
                                mvnLocalRepoDir = "${pwd()}"
                            }

                            println "Maven Local Repository = ${mvnLocalRepoDir}."

                            dir('build') {
                                stage("checkout ${stageKey}") {
                                    checkout scm
                                }

                                stage("build ${stageKey}") {
                                    def jdkTestHome = resolveToolNameToJavaPath(jdkName, mvnName)
                                    println("Setting JDK for testing -Djdk.home=${jdkTestHome}")

                                    def properties = ["\"-Djdk.home=${jdkTestHome}\"", "-Djacoco.skip=true"]

                                    withMaven(jdk: jdkHome, maven: mvnName,
                                        mavenLocalRepo: mvnLocalRepoDir, mavenOpts: '-Xmx512m') {
                                        def script = cmd + options + goals // + properties
                                        bat script.join(' ')
                                    }

                                    lock('maven-surefire-its') {
                                        timeout(time: 15, unit: 'MINUTES') {
                                            withMaven(jdk: jenkinsEnv.jdkFromVersion(os, 8), maven: mvnName,
                                                mavenLocalRepo: mvnLocalRepoDir, mavenOpts: '-Xmx512m',
                                                options: [invokerPublisher()]) {
                                                def script = cmd + optionsITs + goalsITs + propertiesITs
                                                bat script.join(' ')
                                            }
                                        }
                                    }
                                }
                            }
                        } finally {
//                            zip(zipFile: "it--maven-failsafe-plugin--${stageKey}.zip", dir: 'build/maven-failsafe-plugin/target/it', archive: true)
//                            zip(zipFile: "it--surefire-integration-tests--${stageKey}.zip", dir: 'build/surefire-integration-tests/target', archive: true)

                            stage("cleanup ${stageKey}") {
                                // clean up after ourselves to reduce disk space
                                cleanWs()
                            }
                        }
                    }
                } else {
                    node("${env.NIX_LABEL}") {
                        try {
                            println "Basedir = ${pwd()}."

                            def mvnLocalRepoDir

                            dir('.repository') {
                                mvnLocalRepoDir = "${pwd()}"
                            }

                            println "Maven Local Repository = ${mvnLocalRepoDir}."

                            dir('build') {
                                stage("checkout ${stageKey}") {
                                    checkout scm
                                }

                                stage("build ${stageKey}") {
                                    def jdkTestHome = resolveToolNameToJavaPath(jdkName, mvnName)
                                    println("Setting JDK for testing -Djdk.home=${jdkTestHome}")

                                    //https://github.com/jacoco/jacoco/issues/629
                                    def skipJacoco = jdk == '9' ? 'false' : 'true'
                                    def propertiesJdk = "\"-Djdk.home=${jdkTestHome}\""
                                    def properties = [propertiesJdk, "-Djacoco.skip=${skipJacoco}"]
                                    def propertiesITs = [propertiesJdk, '-Djacoco.skip=true']

                                    withMaven(jdk: jdkHome, maven: mvnName,
                                        mavenLocalRepo: mvnLocalRepoDir, mavenOpts: '-Xmx1g') {
                                        def script = cmd + options + goals + properties
                                        sh script.join(' ')
                                    }

                                    lock('maven-surefire-its') {
                                        timeout(time: 15, unit: 'MINUTES') {
                                            withMaven(jdk: jenkinsEnv.jdkFromVersion(os, 8), maven: mvnName,
                                                mavenLocalRepo: mvnLocalRepoDir, mavenOpts: '-Xmx1g',
                                                options: [invokerPublisher()]) {
                                                def script = cmd + optionsITs + goalsITs + propertiesITs
                                                sh script.join(' ')
                                            }
                                        }
                                    }
                                }
                            }
                        } finally {
                            if (indexOfMaven == mavens.size() - 1 && jdk == '9') {
                                jacoco changeBuildStatus: false, execPattern: '**/*.exec', sourcePattern: '**/src/main/java', exclusionPattern: 'pkg/*.class,plexusConflict/*.class,**/surefire570/**/*.class,siblingAggregator/*.class,surefire257/*.class,surefire979/*.class,org/apache/maven/surefire/crb/*.class,org/apache/maven/plugins/surefire/selfdestruct/*.class,org/apache/maven/plugins/surefire/dumppid/*.class,org/apache/maven/plugin/surefire/*.class,org/apache/maven/plugin/failsafe/*.class,jiras/**/*.class,org/apache/maven/surefire/testng/*.class,org/apache/maven/surefire/testprovider/*.class,**/test/*.class,**/org/apache/maven/surefire/group/parse/*.class'
                                junit healthScaleFactor: 0.0, allowEmptyResults: true, keepLongStdio: true, testResults: '**/surefire-integration-tests/target/failsafe-reports/**/*.xml,**/surefire-integration-tests/target/surefire-reports/**/*.xml,**/maven-*/target/surefire-reports/**/*.xml,**/surefire-*/target/surefire-reports/**/*.xml,**/common-*/target/surefire-reports/**/*.xml'
                                if (currentBuild.result == 'UNSTABLE') currentBuild.result = 'FAILURE'
                            }

//                            zip(zipFile: "it--maven-failsafe-plugin--${stageKey}.zip", dir: 'build/maven-failsafe-plugin/target/it', archive: true)
//                            zip(zipFile: "it--surefire-integration-tests--${stageKey}.zip", dir: 'build/surefire-integration-tests/target', archive: true)
//
//                            sh 'tar czvf it1.tgz build/maven-failsafe-plugin/target/it'
//                            sh 'tar czvf it2.tgz build/surefire-integration-tests/target'
//                            archiveArtifacts(artifacts: '**/*.tgz', allowEmptyArchive: true, fingerprint: true, onlyIfSuccessful: false)
//                            archiveArtifacts(artifacts: '*.tgz', allowEmptyArchive: true, fingerprint: true, onlyIfSuccessful: false)

                            stage("cleanup ${stageKey}") {
                                // clean up after ourselves to reduce disk space
                                cleanWs()
                            }
                        }
                    }
                }
            }
        }
    }
}

timeout(time: 18, unit: 'HOURS') {
    try {
        parallel(stages)
        // JENKINS-34376 seems to make it hard to detect the aborted builds
    } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
        // this ambiguous condition means a user probably aborted
        if (e.causes.size() == 0) {
            currentBuild.result = "ABORTED"
        } else {
            currentBuild.result = "FAILURE"
        }
        throw e
    } catch (hudson.AbortException e) {
        // this ambiguous condition means during a shell step, user probably aborted
        if (e.getMessage().contains('script returned exit code 143')) {
            currentBuild.result = "ABORTED"
        } else {
            currentBuild.result = "FAILURE"
        }
        throw e
    } catch (InterruptedException e) {
        currentBuild.result = "ABORTED"
        throw e
    } catch (Throwable e) {
        currentBuild.result = "FAILURE"
        throw e
    } finally {
        stage("notifications") {
            //jenkinsNotify()
        }
    }
}

/**
 * It is used instead of tool(${jdkName}).
 */
def resolveToolNameToJavaPath(jdkToolName, mvnName) {
    def javaHome = null
    try {
        withMaven(jdk: jdkToolName, maven: mvnName) {
            javaHome = isUnix() ? sh(script: 'echo -en $JAVA_HOME', returnStdout: true) : bat(script: '@echo %JAVA_HOME%', returnStdout: true)
        }

        if (javaHome != null) {
            javaHome = javaHome.trim()
            def exec = javaHome + (isUnix() ? '/bin/java' : '\\bin\\java.exe')
            if (!fileExists(exec)) {
                println "The ${exec} does not exist in jdkToolName=${jdkToolName}."
                javaHome = null
            }
        }
    } catch(e) {
        println "Caught an exception while resolving 'jdkToolName' ${jdkToolName} via 'mvnName' ${mvnName}: ${e}"
        javaHome = null;
    }
    assert javaHome != null : "Could not resolve ${jdkToolName} to JAVA_HOME."
    return javaHome
}

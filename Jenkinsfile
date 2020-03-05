#!/usr/bin/env groovy

def buildDockerImage     = "ubuntu:18.04"

def repoName = 'git@github.com:mshenhera/test-conditional-build.git'
def appName          = 'test-conditional-build'
def gitCredentialsId = 'jenkins-github'

// Pipeline job: Commons/java-commons/xxxx-commons
def clientLibName   = 'redis'
def jobParentFolder = env.JOB_NAME.split('/')[0]
def pathInclude     = "${clientLibName}/.*"

properties([
    pipelineTriggers([githubPush()]),
    parameters([
        booleanParam(
            name: 'reload_job',
            defaultValue: false,
            description: 'Reload job arguments without runing any MB actions.')
    ]),
    [
        $class: 'BuildBlockerProperty',
        blockLevel: 'GLOBAL',
        blockingJobs: """^${jobParentFolder}/${appName}/.*""",
        scanQueueFor: 'DISABLE',
        useBuildBlocker: true
    ]
])

pipeline {
    agent none

    // Running Multiple Stages with the Same agent
    // https://jenkins.io/doc/book/pipeline/syntax/#sequential-stages
    stages {
        stage('initial stage') {
            agent any
            stages {
                stage("get source") {
                    steps {
                        script {
                          // Usefull for testing from simple pipeline
                          if ( ! env.BRANCH_NAME ) {
                            env.BRANCH_NAME = 'master'
                          }
                        }

                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: env.BRANCH_NAME]],
                            doGenerateSubmoduleConfigurations: false,
                            extensions: [
                                [$class: 'LocalBranch'],
                                [$class: 'CleanBeforeCheckout'],
                                [
                                    $class: 'PathRestriction',
                                    excludedRegions: '',
                                    includedRegions: "${pathInclude}"
                                ],
                                [$class: 'UserExclusion', excludedUsers: '''mshenhera'''
                                ]
                            ],
                            submoduleCfg: [],
                            userRemoteConfigs: [[url: repoName, credentialsId: gitCredentialsId]]
                        ])

                        script {
                            if ( env.BUILD_NUMBER == "1" || params.reload_job ) {
                              currentBuild.result = 'NOT_BUILT'
                              env.CONTINUE_BUILD = 'false'
                              return
                            }
                            env.CONTINUE_BUILD = 'true'
                            
                            shortCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                            commitCount = sh(returnStdout: true, script: 'git rev-list --count HEAD').trim()
                
                            currentBuild.displayName = "${commitCount}-${shortCommit}-${BUILD_NUMBER}"
                            gitCommitAuthor = sh(returnStdout: true, script: 'git show --format="%aN" ${gitCommit} | head -1').trim()
                        }
                        
                        sh '''
                            env
                            mkdir -p ${JENKINS_HOME}/gradle-cache/${JOB_NAME}
                            echo "Test Me!" > ${JENKINS_HOME}/gradle-cache/${JOB_NAME}/test.me
                        '''
                    }
                }

                stage("second step") {
                    when { environment name: 'CONTINUE_BUILD', value: 'true' }
                    agent {
                        docker {
                            reuseNode true
                            image buildDockerImage
                            args "-v ${env.JENKINS_HOME}/gradle-cache/${env.JOB_NAME}:/gradle \
                                  -e JAVA_APP_NAME=${clientLibName}"
                        }
                    }
                    steps {
                        
                                echo "inside docker"

                    }
                }
           }
        }
    }
}

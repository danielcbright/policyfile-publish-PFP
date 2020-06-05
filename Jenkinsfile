def POLICY_NAME = params.POLICY_NAME
def POLICY_ID = params.POLICY_ID
def POLICY_ARCHIVE = "${POLICY_NAME}-${POLICY_ID}.tgz"
def POLICY_GROUPS_TXT
def POLICY_GROUPS = [:]

pipeline {
  agent any
  environment {
    HOME = "/root/"
  }
  stages {
    stage('Download Files') {
      parallel {
        stage('Download Files') {
          steps {
            sh '''
              mkdir gcs-files
              mkdir s3-files
              mkdir azure-files
            '''
          }
        }

        stage('Download from GCS') {
          steps {
            dir(path: 'gcs-files') {
              googleStorageDownload(credentialsId: 'gcs-policyfile-archive', bucketUri: "gs://policyfile-archive/$POLICY_NAME/$POLICY_ID/*", localDirectory: './', pathPrefix: "$POLICY_NAME/$POLICY_ID/")
            }

          }
        }

        stage('Download from S3') {
          steps {
            dir(path: 's3-files') {
              withAWS(credentials: 'aws-policyfile-archive', region: 'us-east-1') {
                s3Download(file: "$POLICY_ARCHIVE", bucket: 'dcb-policyfile-archive', path: "$POLICY_NAME/$POLICY_ID/$POLICY_ARCHIVE")
                s3Download(file: 'policy_groups.txt', bucket: 'dcb-policyfile-archive', path: "$POLICY_NAME/$POLICY_ID/policy_groups.txt")  
              }
            }
          }
        }

        stage('Download from Azure') {
          steps {
            dir(path: 'azure-files') {
              azureDownload(containerName: 'policyfile-archive', flattenDirectories: true, downloadType: 'container', includeFilesPattern: "$POLICY_NAME/$POLICY_ID/**", storageCredentialId: 'fbc18e3a-1207-4a90-9f29-765a8b88ac86')
            }
          }
        }
      }
    }

    // We'll work out of the GCS files since the download steps are just for demonstration purposes, change the directory to whatever storage you are using.
    stage('Prep Policy Group Stages') {
      steps {
        dir ("gcs-files") {
          script {
            POLICY_GROUPS_TXT = sh (
              script: '/usr/bin/cat policy_groups.txt',
              returnStdout: true
            ).trim()
            POLICY_GROUPS = POLICY_GROUPS_TXT.split('\n')
          }
        }
      }
    }

    stage('Push Policy') {
      steps {
        wrap([$class: 'ChefIdentityBuildWrapper', jobIdentity: 'jenkins-dbright']) {
          dir ("gcs-files") {
            script {
              echo "Iterating policy_groups.txt in top->bottom order..."
              echo "${POLICY_GROUPS_TXT}"
              for (GROUP in POLICY_GROUPS) {
                def VARS = GROUP.split(':')
                if ( "${VARS[1]}" == "auto" ) {
                  echo "POLICY_GROUP: ${VARS[0]} set to auto approve, running push-archive now"
                  sh "/opt/chef-workstation/bin/chef push-archive ${VARS[0]} $POLICY_ARCHIVE"
                } else if ( "${VARS[1]}" == "manual" ) {
                  def userInputPushArchive = input (
                                            message: "Publish ${POLICY_NAME} to ${VARS[0]}?",
                                            parameters: [
                                              choice(
                                                name: 'Push-archive',
                                                choices: ['no', 'yes'].join('\n'),
                                                description: "Choose \"yes\" to publish $POLICY_ARCHIVE to ${VARS[0]}"
                                              )
                                            ])
                  if ("${userInputPushArchive}" == "yes") {
                    echo "POLICY_GROUP: ${VARS[0]} set to manual approve, approved and running push-archive now"
                    sh "/opt/chef-workstation/bin/chef push-archive ${VARS[0]} $POLICY_ARCHIVE"
                  } else if ( "${userInputPushArchive}" == "no" ) {
                    echo "Not pushing based on input"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}

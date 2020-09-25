/* groovylint-disable NestedBlockDepth */
/* groovylint-disable-next-line CompileStatic */
def chefWrapperId = 'ChefIdentityBuildWrapper'
def chefJobId = 'jenkins-dbright'
def policyGroupsTxt
def policyGroups = [:]
def gcsDownloadDir = 'gcs-files'
def s3DownloadDir = 's3-files'
def azureStorageCredentialsId = 'fbc18e3a-1207-4a90-9f29-765a8b88ac86'
def azureContainerName = 'policyfile-archive'
def azureDownloadDir = 'azure-files'
def gcsCredentialsId = 'gcs-policyfile-archive'
def gcsBucket = 'gs://policyfile-archive'
def awsWrapperId = 'aws-policyfile-archive'
def awsWrapperRegion = 'us-east-1'
def s3Bucket = 'dcb-policyfile-archive'

pipeline {
  agent any
  parameters {
        string(defaultValue: 'NOTDEFINED', name: 'policyName', description: 'policyName')
        string(defaultValue: 'NOTDEFINED', name: 'policyId', description: 'policyId')
    }
  environment {
    HOME = '/root/'
  }
  stages {
    stage('Download Files') {
      parallel {
        stage('Create Download Directories') {
          steps {
            sh """
              mkdir $gcsDownloadDir
              mkdir $s3DownloadDir
              mkdir $azureDownloadDir
            """
          }
        }
        stage('Download from GCS') {
          steps {
            dir(path: "$gcsDownloadDir") {
              googleStorageDownload(
                credentialsId: "$gcsCredentialsId",
                bucketUri: "$gcsBucket/${params.policyName}/${params.policyId}/*",
                localDirectory: './',
                pathPrefix: "${params.policyName}/${params.policyId}/")
            }
          }
        }
        stage('Download from S3') {
          steps {
            dir(path: "$s3DownloadDir") {
              withAWS(credentials: "$awsWrapperId", region: "$awsWrapperRegion") {
                s3Download(
                  file: "${params.policyName}-${params.policyId}.tgz",
                  bucket: "$s3Bucket",
                  path: "${params.policyName}/${params.policyId}/${params.policyName}-${params.policyId}.tgz"
                )
                s3Download(
                  file: 'policy_groups.txt',
                  bucket: "$s3Bucket",
                  path: "${params.policyName}/${params.policyId}/policy_groups.txt"
                )
              }
            }
          }
        }

        stage('Download from Azure') {
          steps {
            dir(path: "$azureDownloadDir") {
              azureDownload(
                containerName: "$azureContainerName",
                flattenDirectories: true,
                downloadType: 'container',
                includeFilesPattern: "${params.policyName}/${params.policyId}/**",
                storageCredentialId: "$azureStorageCredentialsId"
              )
            }
          }
        }
      }
    }
    // We'll work out of the GCS files since the download steps are just for
    // demonstration purposes, change the directory to whatever storage you are using.
    stage('Prep Policy Group Stages') {
      steps {
        dir ("$gcsDownloadDir") {
          script {
            policyGroupsTxt = sh (
              script: '/usr/bin/cat policy_groups.txt',
              returnStdout: true
            ).trim()
            policyGroups = policyGroupsTxt.split('\n')
          }
        }
      }
    }
    stage('Push Policy') {
      steps {
        wrap([$class: "$chefWrapperId", jobIdentity: "$chefJobId"]) {
          dir ("$gcsDownloadDir") {
            script {
              echo 'Iterating policy_groups.txt in top->bottom order...'
              echo "$policyGroupsTxt"
              for (group in policyGroups) {
                vars = group.split(':')
                policyGroup = "${vars[0]}"
                policyDeploy = "${vars[1]}"
                if ( "$policyDeploy" == 'auto' ) {
                  echo "POLICY_GROUP: $policyGroup set to auto approve, running push-archive now"
                  sh "/opt/chef-workstation/bin/chef push-archive $policyGroup ${params.policyName}-${params.policyId}.tgz"
                } else if ( "$policyDeploy" == 'manual' ) {
                  /* groovylint-disable-next-line NoDef, VariableTypeRequired */
                  userInputPushArchive = input (
                    message: "Publish ${params.policyName} to $policyGroup?",
                    parameters: [
                      choice(
                        name: 'Push-archive',
                        /* groovylint-disable-next-line DuplicateStringLiteral */
                        choices: ['no', 'yes'].join('\n'),
                        description: "Choose \"yes\" to publish ${params.policyName}-${params.policyId}.tgz to $policyGroup"
                        )
                    ]
                  )
                  /* groovylint-disable-next-line DuplicateStringLiteral */
                  if ("$userInputPushArchive" == 'yes') {
                    echo "POLICY_GROUP: $policyGroup set to manual approve, approved and running push-archive now"
                    sh "/opt/chef-workstation/bin/chef push-archive $policyGroup ${params.policyName}-${params.policyId}.tgz"
                  /* groovylint-disable-next-line DuplicateStringLiteral */
                  } else if ( "$userInputPushArchive" == 'no' ) {
                    echo 'Not pushing based on input'
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

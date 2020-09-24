/* groovylint-disable NestedBlockDepth */
/* groovylint-disable-next-line CompileStatic */
String policyName = params.policyName
String policyId = params.policyId
String chefWrapperId = 'ChefIdentityBuildWrapper'
String chefJobId = 'jenkins-dbright'
String policyArchive = "${policyName}-${policyId}.tgz"
String policyGroupsTxt
String policyGroups = [:]
String gcsDownloadDir = 'gcs-files'
String s3DownloadDir = 's3-files'
String azureStorageCredentialsId = 'fbc18e3a-1207-4a90-9f29-765a8b88ac86'
String azureContainerName = 'policyfile-archive'
String azureDownloadDir = 'azure-files'
String gcsCredentialsId = 'gcs-policyfile-archive'
String gcsBucket = 'gs://policyfile-archive'
String awsWrapperId = 'aws-policyfile-archive'
String awsWrapperRegion = 'us-east-1'
String s3Bucket = 'dcb-policyfile-archive'

pipeline {
  agent any
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
                bucketUri: "$gcsBucket/$policyName/$policyId/*",
                localDirectory: './',
                pathPrefix: "$policyName/$policyId/")
            }
          }
        }

        stage('Download from S3') {
          steps {
            dir(path: "$s3DownloadDir") {
              withAWS(credentials: "$awsWrapperId", region: "$awsWrapperRegion") {
                s3Download(
                  file: "$policyArchive",
                  bucket: "$s3Bucket",
                  path: "$policyName/$policyId/$policyArchive"
                )
                s3Download(
                  file: 'policy_groups.txt',
                  bucket: "$s3Bucket",
                  path: "$policyName/$policyId/policy_groups.txt"
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
                includeFilesPattern: "$policyName/$policyId/**",
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
                String policyGroup = "${vars[0]}"
                String policyDeploy = "${vars[1]}"
                if ( "$policyDeploy" == 'auto' ) {
                  echo "POLICY_GROUP: $policyGroup set to auto approve, running push-archive now"
                  sh "/opt/chef-workstation/bin/chef push-archive $policyGroup $policyArchive"
                } else if ( "$policyDeploy" == 'manual' ) {
                  /* groovylint-disable-next-line NoDef, VariableTypeRequired */
                  def userInputPushArchive = input (
                    message: "Publish ${policyName} to $policyGroup?",
                    parameters: [
                      choice(
                        name: 'Push-archive',
                        /* groovylint-disable-next-line DuplicateStringLiteral */
                        choices: ['no', 'yes'].join('\n'),
                        description: "Choose \"yes\" to publish $policyArchive to $policyGroup"
                        )
                    ]
                  )
                  /* groovylint-disable-next-line DuplicateStringLiteral */
                  if ("$userInputPushArchive" == 'yes') {
                    echo "POLICY_GROUP: $policyGroup set to manual approve, approved and running push-archive now"
                    sh "/opt/chef-workstation/bin/chef push-archive $policyGroup $policyArchive"
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

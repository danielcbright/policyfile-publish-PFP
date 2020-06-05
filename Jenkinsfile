pipeline {
  agent any
  stages {
    stage('Download Files') {
      parallel {
        stage('Download Files') {
          steps {
            sh '''mkdir gcs-files
mkdir s3-files
mkdir azure-files'''
          }
        }

        stage('Download from GCS') {
          steps {
            dir(path: 'gcs-files') {
              googleStorageDownload(credentialsId: 'gcs-policyfile-archive', bucketUri: 'gs://policyfile-archive/linux-base/4826c36e38d4ba4e0a614c26f673be06bd884a60e23cbbbf3aef04ee0e6b03f1/*', localDirectory: './', pathPrefix: 'linux-base/4826c36e38d4ba4e0a614c26f673be06bd884a60e23cbbbf3aef04ee0e6b03f1/')
            }

          }
        }

        stage('Download from S3') {
          steps {
            dir(path: 's3-files') {
              withAWS(credentials: 'aws-policyfile-archive', region: 'us-east-1') {
                s3Download(file: 'linux-base-4826c36e38d4ba4e0a614c26f673be06bd884a60e23cbbbf3aef04ee0e6b03f1.tgz', bucket: 'dcb-policyfile-archive', path: 'linux-base/4826c36e38d4ba4e0a614c26f673be06bd884a60e23cbbbf3aef04ee0e6b03f1/linux-base-4826c36e38d4ba4e0a614c26f673be06bd884a60e23cbbbf3aef04ee0e6b03f1.tgz')
                              s3Download(file: 'policy_groups.txt', bucket: 'dcb-policyfile-archive', path: 'linux-base/4826c36e38d4ba4e0a614c26f673be06bd884a60e23cbbbf3aef04ee0e6b03f1/policy_groups.txt')

                  
              }

            }

          }
        }

        stage('Download from Azure') {
          steps {
            dir(path: 'azure-files') {
              azureDownload(containerName: 'policyfile-archive', flattenDirectories: true, downloadType: 'container', includeFilesPattern: 'linux-base/ebc5bf9f3cd5f215b5b236bdcd2c0e9cc905ae591e1826776ef98040040d189f/**', storageCredentialId: 'fbc18e3a-1207-4a90-9f29-765a8b88ac86')
                            }
          }
        }
      }
    }
  }
}

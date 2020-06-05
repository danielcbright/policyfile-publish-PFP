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
              googleStorageDownload(credentialsId: 'gcs-policyfile-archive', bucketUri: 'gs://policyfile-archive/linux-base/ebc5bf9f3cd5f215b5b236bdcd2c0e9cc905ae591e1826776ef98040040d189f/', localDirectory: './', pathPrefix: '*.*')
            }

          }
        }

        stage('Download from S3') {
          steps {
            dir(path: 's3-files') {
              withAWS(credentials: 'aws-policyfile-archive', region: 'us-east-1') {
                s3Download(file: '*.*', bucket: 'dcb-policyfile-archive', path: 'policyfile-archive/linux-base/ebc5bf9f3cd5f215b5b236bdcd2c0e9cc905ae591e1826776ef98040040d189f/')
              }

            }

          }
        }

        stage('Download from Azure') {
          steps {
            dir(path: 'azure-files')
          }
        }

      }
    }

  }
}
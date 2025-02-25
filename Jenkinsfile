pipeline {
    agent any
    tools {
        maven 'Maven3'
    }
    parameters {
        choice(name: 'DEPLOY_ACTION', choices: ['DEPLOY_NEW', 'ROLLBACK'], description: 'Choose DEPLOY_NEW for normal deployment or ROLLBACK for rollback')
    }
    environment {
        AWS_REGION = 'ap-south-1'
        EB_APP_NAME = 'MyApp'
        EB_ENV_NAME = 'Test-env-jenkins-env'
        WAR_STORAGE_PATH = '/var/lib/jenkins/war_backups'
        S3_BUCKET = 'flipkart-backup-jenkins-stagging'
        BUILD_LIMIT = 20
        WAR_COUNT_LIMIT = 3
    }
    stages {
        stage('Checkout Code') {
            when {
                expression { params.DEPLOY_ACTION == 'DEPLOY_NEW' }
            }
            steps {
                git branch: 'master', url: 'https://github.com/Shradha3001/api-test-repo.git'
            }
        }
        stage('Build WAR File') {
            when {
                expression { params.DEPLOY_ACTION == 'DEPLOY_NEW' }
            }
            steps {
                sh 'mvn clean package -Dmaven.test.skip=true'

                script {
                    def warFile = sh(script: "ls target/*.war", returnStdout: true).trim()
                    sh "cp ${warFile} ${WAR_STORAGE_PATH}/app_${env.BUILD_NUMBER}.war"
                }
            }
        }
        stage('Manage WAR Storage') {
            steps {
                script {
                    sh """
                        ls -t ${WAR_STORAGE_PATH}/app_*.war | tail -n +${WAR_COUNT_LIMIT+1} | xargs rm -f
                    """
                    def latestWar = sh(script: "ls -t ${WAR_STORAGE_PATH}/app_*.war | head -n 1", returnStdout: true).trim()
                    sh "aws s3 cp ${latestWar} s3://${S3_BUCKET}/prod/"
                }
            }
        }
        stage('Deploy to Elastic Beanstalk') {
            steps {
                script {
                    def warFileToDeploy = params.DEPLOY_ACTION == 'ROLLBACK' ? "${WAR_STORAGE_PATH}/last_successful.war" : sh(script: "ls target/*.war", returnStdout: true).trim()
                    sh "aws s3 cp ${warFileToDeploy} s3://${S3_BUCKET}/prod/app.war"
                    sh "aws elasticbeanstalk update-environment --application-name ${env.EB_APP_NAME} --environment-name ${env.EB_ENV_NAME} --version-label v${env.BUILD_NUMBER} --region ${env.AWS_REGION}"
                }
            }
        }
    }
}
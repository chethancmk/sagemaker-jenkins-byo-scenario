pipeline {

    agent any

    environment {
        AWS_ECR_LOGIN = 'true'
        DOCKER_CONFIG= "${params.JENKINSHOME}"
        AWS_REGION= "${params.AWS_REGION}"
    }

    stages {
        stage("Checkout") {
            steps {
               checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/chethancmk/sagemaker-jenkins-byo-scenario']]])
            }
        }

        stage("BuildPushContainer") {
            steps {
              sh """
                echo "${params.ECRURI}"
                aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${params.ECRURI}
 	              docker build -t scikit-byo:${env.BUILD_ID} .
                docker tag scikit-byo:${env.BUILD_ID} ${params.ECRURI}:${env.BUILD_ID} 
                docker push ${params.ECRURI}:${env.BUILD_ID}
                echo ${params.S3_PACKAGED_LAMBDA}
              """
            }
        }
        
        stage("TrainModel") {
            steps { 
              sh """
               aws sagemaker create-training-job --training-job-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID} --algorithm-specification TrainingImage="${params.ECRURI}:${env.BUILD_ID}",TrainingInputMode="File" --role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST} --input-data-config '{"ChannelName": "training", "DataSource": { "S3DataSource": { "S3DataType": "S3Prefix", "S3Uri": "${params.S3_TRAIN_DATA}"}}}' --resource-config InstanceType='ml.c4.2xlarge',InstanceCount=1,VolumeSizeInGB=5 --output-data-config S3OutputPath='${S3_MODEL_ARTIFACTS}' --stopping-condition MaxRuntimeInSeconds=3600 --region='${AWS_REGION}'
              """
             }
        }

      stage("TrainStatus") {
            steps  {
                        script {
                            timeout(10) {
                                waitUntil(initialRecurrencePeriod: 15000) {
                                   def response = sh """        
                    /usr/local/bin/aws lambda invoke --function-name ${params.LAMBDA_CHECK_STATUS_TRAINING} --cli-binary-format raw-in-base64-out --region us-east-1 --payload '{"TrainingJobName": "${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}"}' response.json                
                    """
                                    println  response
                                     return(true)
                                }
                            }
                        }
                    }
      }

      stage("DeployToTest") {
            steps { 
              sh """
               aws sagemaker create-model --model-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --primary-container ContainerHostname=${env.BUILD_ID},Image=${params.ECRURI}:${env.BUILD_ID},ModelDataUrl='${S3_MODEL_ARTIFACTS}'/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}/output/model.tar.gz,Mode='SingleModel' --execution-role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST} --region='${AWS_REGION}'
               aws sagemaker create-endpoint-config --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --production-variants VariantName='single-model',ModelName=${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test,InstanceType='ml.m4.xlarge',InitialVariantWeight=1,InitialInstanceCount=1 --region='${AWS_REGION}'
               aws sagemaker create-endpoint --endpoint-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Test --region='${AWS_REGION}'
               sleep 300
              """
             }
        }

      stage("SmokeTest") {
            steps { 
              script {
                 def response = sh """ 
                  sleep 30
              """
              }
            }
        }

      stage("DeployToProd") {
            steps { 
              sh """
               aws sagemaker create-model --model-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --primary-container ContainerHostname=${env.BUILD_ID},Image=${params.ECRURI}:${env.BUILD_ID},ModelDataUrl='${S3_MODEL_ARTIFACTS}'/${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}/output/model.tar.gz,Mode='SingleModel' --execution-role-arn ${params.SAGEMAKER_EXECUTION_ROLE_TEST} --region='${AWS_REGION}'
               aws sagemaker create-endpoint-config --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --production-variants VariantName='single-model',ModelName=${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod,InstanceType='ml.m4.xlarge',InitialVariantWeight=1,InitialInstanceCount=1 --region='${AWS_REGION}'
               aws sagemaker create-endpoint --endpoint-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --endpoint-config-name ${params.SAGEMAKER_TRAINING_JOB}-${env.BUILD_ID}-Prod --region='${AWS_REGION}'
              """
             }
        }

  }
}   

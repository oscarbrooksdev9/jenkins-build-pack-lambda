#!groovy

import groovy.json.JsonOutput
import groovy.json.JsonSlurper
import groovy.transform.Field

// To be replaced as @Field def repo_credential_id = "value" for repo_credential_id, repo_base and repo_core

@Field def repo_credential_id = "jazz_repocreds"
@Field def repo_base = "18.235.7.234"
@Field def repo_core = "slf"
@Field def scm_type = "gitlab"

@Field def configModule
@Field def configLoader
@Field def scmModule
@Field def serviceConfigdata
@Field def events
@Field def lambdaEvents
@Field def serviceMetadataLoader
@Field def utilModule
@Field def environmentDeploymentMetadata
@Field def sonarModule

@Field def auth_token = ''
@Field def g_base_url = ''
@Field def g_svc_admin_cred_ID = 'SVC_ADMIN'
@Field def current_environment = ''
@Field def isLambdaUpdateRequired = false

node() {

  def jazzBuildModuleURL = getBuildModuleUrl()
  loadBuildModules(jazzBuildModuleURL)

  def jazz_prod_api_id = utilModule.getAPIIdForCore(configLoader.AWS.API["PROD"])
  g_base_url = "https://${jazz_prod_api_id}.execute-api.${configLoader.AWS.REGION}.amazonaws.com/prod"

  echo "Build triggered via branch: " + params.scm_branch + " with params " + params
  def _event = ""

  def branch = params.scm_branch
  def repo_name = params.service_name
  def gitCommitOwner
  def gitCommitHash
  def context_map
  def environment_logical_id

  if (domain && domain != "") {
    repo_name = params.domain + "_" + params.service_name
  }

  def config
  def domain = params.domain

  stage('Checkout code base') {

    sh 'rm -rf ' + repo_name
    sh 'mkdir ' + repo_name
    sh 'pwd'

    def repocloneUrl
    if (domain && domain == "jazz") {
      repocloneUrl = scmModule.getCoreRepoCloneUrl(repo_name)
    } else {
      auth_token = setCredentials()
      repocloneUrl = scmModule.getRepoCloneUrl(repo_name)
    }

    dir(repo_name) {
      checkout([$class: 'GitSCM', branches: [
        [name: '*/' + params.scm_branch]
      ], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
          [credentialsId: repo_credential_id, url: repocloneUrl]
        ]])
    }

    def configObj = dir(repo_name) {
      LoadConfiguration()
    }

    if (configObj.service_id) {
      config = serviceMetadataLoader.loadServiceMetadata(configObj.service_id)
    } else {
      error "Service Id is not available."
    }
  }

  if (!config) {
    error "Failed to fetch service metadata from catalog"
  }

  dir(repo_name) {
    scmModule.setServiceConfig(config)
    def envApi = "${g_base_url}/jazz/environments"
    environmentDeploymentMetadata.initialize(config, configLoader, scmModule, branch, env.BUILD_URL, env.BUILD_ID, envApi, auth_token)
    gitCommitHash = scmModule.getRepoCommitHash()
    gitCommitOwner = scmModule.getRepoCommitterInfo(gitCommitHash)
    context_map = [created_by: config['created_by'], deployed_by: gitCommitOwner]
    environment_logical_id = '';

    if (branch == 'master') {
      current_environment = 'prod'
      environment_logical_id = 'prod';
    } else {
      current_environment = 'dev'
      environment_logical_id = environmentDeploymentMetadata.getEnvironmentLogicalId();
    }

    if (!environment_logical_id && config['domain'] != 'jazz') {
      error "The environment has not been created yet and its missing the environment id"
    }
  }
  if (!events) {
    error "Can't load events module"
  } //Fail here
  def eventsApi = "${g_base_url}/jazz/events"
  events.initialize(configLoader, config, "SERVICE_DEPLOYMENT", branch, environment_logical_id, eventsApi)

  def runtime = config['providerRuntime']
  def service = config['service']
  def isScheduleEnabled = isEnabled(config, "eventScheduleRate")
  def isStreamEnabled = isEnabled(config, "event_source_kinesis")
  def isDynamoDbEnabled = isEnabled(config, "event_source_dynamodb")
  def isS3EventEnabled = isEnabled(config, "event_source_s3")
  def isSQSEventEnabled = isEnabled(config, "event_source_sqs")
  def isEc2EventEnabled = isEnabled(config, "event_source_ec2")
  def isEventSchdld = false
  def internalAccess = config['require_internal_access']
  domain = config['domain']
  def roleARN = configLoader.AWS.ROLEID.replaceAll("/", "\\\\/")
  // @TODO: the below statement will be replaced with regular expression in very near future;
  def roleId = roleARN.substring(roleARN.indexOf("::") + 2, roleARN.lastIndexOf(":"))

  sonarModule.initialize(configLoader, config, branch)
  stackName = "${configLoader.INSTANCE_PREFIX}-${config['domain']}-${config['service']}-${environment_logical_id}"

  if (isScheduleEnabled || isEc2EventEnabled || isDynamoDbEnabled || isSQSEventEnabled || isStreamEnabled) {
    isEventSchdld = true
  }

  def requestId = utilModule.generateRequestId()
  if (requestId != null) {
    events.setRequestId(requestId)
    environmentDeploymentMetadata.setRequestId(requestId)
  } else {
    error "Request Id Generation failed"
  }

  events.sendStartedEvent('CREATE_DEPLOYMENT', 'Function deployment started', environmentDeploymentMetadata.generateDeploymentMap("started", environment_logical_id, gitCommitHash), environment_logical_id)

  dir(repo_name) {
    loadServerlessConfig(runtime, isEventSchdld, isScheduleEnabled, isEc2EventEnabled, isS3EventEnabled, isSQSEventEnabled, isStreamEnabled, isDynamoDbEnabled)
    if (domain && domain == "jazz") {
      stage('Update Service Template') {
        echo "Inside Update service template"
        try {
          updatePlatformServiceTemplate(config, roleARN, roleId, current_environment, repo_name)
        } catch (error) {
          error "Error occured while updating service template."
        }
      }
    }
    stage('Pre-Build Validation') {
      events.sendStartedEvent('VALIDATE_PRE_BUILD_CONF', 'pre-build validation started', context_map, environment_logical_id)
      try {
        send_status_email(config, 'STARTED', "")
        validateDeploymentConfigurations(config)
      } catch (ex) {
        send_status_email(config, 'FAILED', "")
        events.sendFailureEvent('VALIDATE_PRE_BUILD_CONF', ex.getMessage(), context_map, environment_logical_id)
        events.sendFailureEvent('UPDATE_DEPLOYMENT', ex.getMessage(), environmentDeploymentMetadata.generateDeploymentMap("failed", environment_logical_id, gitCommitHash), environment_logical_id)
        error ex.getMessage()
      }
      events.sendCompletedEvent('VALIDATE_PRE_BUILD_CONF', 'pre-build validation completed', context_map, environment_logical_id)
    }

    if (config['domain'] && config['domain'] != 'jazz' && configLoader.CODE_QUALITY.SONAR.ENABLE_SONAR == "true") {
      stage('Code Quality Check') {
        events.sendStartedEvent('CODE_QUALITY_CHECK', 'code quality check starts', context_map, environment_logical_id)
        try {
          runValidation(runtime)
          sonarModule.doAnalysis()
        } catch (ex) {
          events.sendFailureEvent('CODE_QUALITY_CHECK', ex.getMessage(), context_map, environment_logical_id)
          events.sendFailureEvent('UPDATE_DEPLOYMENT', ex.getMessage(), environmentDeploymentMetadata.generateDeploymentMap("failed", environment_logical_id, gitCommitHash), environment_logical_id)
          error ex.getMessage()
        }
        events.sendCompletedEvent('CODE_QUALITY_CHECK', 'code quality check completed', context_map, environment_logical_id)
      }
    }

    stage('Build') {
      events.sendStartedEvent('BUILD', 'build starts', context_map, environment_logical_id)
      try {
        buildLambda(runtime)
      } catch (ex) {
        events.sendFailureEvent('BUILD', ex.getMessage(), context_map, environment_logical_id)
        events.sendFailureEvent('UPDATE_DEPLOYMENT', ex.getMessage(), environmentDeploymentMetadata.generateDeploymentMap("failed", environment_logical_id, gitCommitHash), environment_logical_id)
        error ex.getMessage()
      }
      events.sendCompletedEvent('BUILD', 'build completed', context_map, environment_logical_id)
    }

    def env_key
    if (branch == "master") {
      env_key = "PROD"
    } else {
      env_key = "DEV"
    }
    stage("Deployment to ${env_key} environment") {
      events.sendStartedEvent('DEPLOY_TO_AWS', 'Deployment started to staging AWS environment', context_map, environment_logical_id)
      events.sendStartedEvent('UPDATE_ENVIRONMENT', "Environment status update event for ${env_key} deployment", environmentDeploymentMetadata.generateEnvironmentMap("deployment_started", environment_logical_id, null), environment_logical_id)

      echo "starts Deployment to ${env_key} Env"
      def lambdaARN = null
      environmentDeploymentMetadata.setEnvironmentEndpoint(lambdaARN)

      events.sendStartedEvent('UPDATE_ENVIRONMENT', "Environment status update event for ${env_key} deployment", environmentDeploymentMetadata.generateEnvironmentMap("deployment_started", environment_logical_id, null), environment_logical_id)

      withCredentials([
        [$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: configLoader.AWS_CREDENTIAL_ID, secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
      ]) {
        try {
          // initialize aws credentials
          sh "aws configure set profile.cloud-api.region ${configLoader.AWS.REGION}"
          sh "aws configure set profile.cloud-api.aws_access_key_id $AWS_ACCESS_KEY_ID"
          sh "aws configure set profile.cloud-api.aws_secret_access_key $AWS_SECRET_ACCESS_KEY"

          loadServerlessConfig(runtime, isEventSchdld, isScheduleEnabled, isEc2EventEnabled, isS3EventEnabled, isSQSEventEnabled, isStreamEnabled, isDynamoDbEnabled)

          // Generate serverless yml file with domain added in function name
          echo "Generate deployment env with domain"

          if (isScheduleEnabled) {
            def eventsArns = getEventsArn(config, environment_logical_id)
            for (def i = 0; i < eventsArns.size(); i++) {
              def arn = eventsArns[i]
              events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", arn, "cloudwatch_event", config), environment_logical_id);
            }
          }

          writeServerlessFile(config, environment_logical_id)

          def envBucketKey = "${env_key}${configLoader.JAZZ.S3_BUCKET_NAME_SUFFIX}"
          echoServerlessFile()
          def deployOutput = servelessDeploy(environment_logical_id, envBucketKey)

          if (deployOutput != 'success') {
            if (config['event_source_s3']) {
              handleS3BucketError(deployOutput, config['event_source_s3'], environment_logical_id, envBucketKey, isEventSchdld)
            } else if (config['event_source_sqs']) {
              handleSqsError(deployOutput, config, environment_logical_id, envBucketKey)
            } else if (config['event_source_kinesis']) {
              handleKinesisStreamError(deployOutput, config['event_source_kinesis'], environment_logical_id, envBucketKey)
            } else if (config['event_source_dynamodb']) {
              handleDynamoDbStreamError(deployOutput, config, environment_logical_id, envBucketKey)
            } else {
              echo "Exception occured while serverless deployment to ${environment_logical_id} environment : $deployOutput"
              error "Exception occured while serverless deployment to ${environment_logical_id} environment"
            }
          } else {
            if (config['event_source_s3']) {
              def bucket_arn = "arn:aws:s3:::${config['event_source_s3']}"
              def s3_bucket_arn = lambdaEvents.getEventResourceNamePerEnvironment(bucket_arn, environment_logical_id, "-")
              events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", s3_bucket_arn, "s3", config), environment_logical_id);
            } else if (config['event_source_sqs']) {
              def event_source_sqs_arn = lambdaEvents.getEventResourceNamePerEnvironment(config['event_source_sqs'], environment_logical_id, "_")
              events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", event_source_sqs_arn, "sqs", config), environment_logical_id);
            } else if (config['event_source_kinesis']) {
              def event_source_kinesis_stream_arn = lambdaEvents.getEventResourceNamePerEnvironment(config['event_source_kinesis'], environment_logical_id, "_")
              events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", event_source_kinesis_stream_arn, "kinesis_stream", config), environment_logical_id);
            } else if (config['event_source_dynamodb']) {
              def dynamodbTableName = lambdaEvents.splitAndGetResourceName(config['event_source_dynamodb'], environment_logical_id)
              def stream_details = lambdaEvents.getDynamoDbStreamDetails(dynamodbTableName)
              def event_source_dynamodb_arn = lambdaEvents.getEventResourceNamePerEnvironment(config['event_source_dynamodb'], environment_logical_id, "_")
              events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", event_source_dynamodb_arn, "dynamodb", config), environment_logical_id);
              events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", stream_details.StreamArn, "dynamodb_stream", config), environment_logical_id);
            }
          }

          def function_arn = getLambdaARN(stackName);
          lambdaARN = function_arn.split(":(?!.*:.*)")[0]
          events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", lambdaARN, "lambda", config), environment_logical_id);

          echo "lambdaARN: ${lambdaARN}"

          if (isLambdaUpdateRequired) {
            def event_source_s3 = lambdaEvents.getEventResourceNamePerEnvironment(config['event_source_s3'], environment_logical_id, "-")
            lambdaEvents.updateLambdaPermissionAndNotification(lambdaARN, event_source_s3, config['event_action_s3'])
          }

          if (config['event_source_dynamodb'] || config['event_source_sqs']) {
            def role_name = "${configLoader.INSTANCE_PREFIX}-${config['domain']}-${config['service']}-${environment_logical_id}"
            def role_arn = lambdaEvents.getRoleArn(role_name)
            events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", role_arn, "iam_role", config), environment_logical_id);
          }

          // reset Credentials
          resetCredentials()
          if (domain != "jazz") {
            createSubscriptionFilters(stackName, configLoader.AWS.REGION, roleId);
          }

          if (domain && domain == "jazz") {
            serviceConfigdata.setLogStreamPermission(config)
            serviceConfigdata.setKinesisStream(config)
          }
          def svc_response = echoServiceInfo(environment_logical_id)
          send_status_email(config, 'COMPLETED', svc_response)

        } catch (ex) {
          send_status_email(config, 'FAILED', '')
          events.sendFailureEvent('UPDATE_ENVIRONMENT', ex.getMessage(), environmentDeploymentMetadata.generateEnvironmentMap("deployment_failed", environment_logical_id, null), environment_logical_id)
          events.sendFailureEvent('UPDATE_DEPLOYMENT', ex.getMessage(), environmentDeploymentMetadata.generateDeploymentMap("failed", environment_logical_id, gitCommitHash), environment_logical_id)
          events.sendFailureEvent('DEPLOY_TO_AWS', ex.getMessage(), context_map, environment_logical_id)
          error ex.getMessage()
        }
        environmentDeploymentMetadata.setEnvironmentEndpoint(lambdaARN)
        def serviceContext = [created_by: config['created_by'], deployed_by: gitCommitOwner]

        events.sendCompletedEvent('UPDATE_ENVIRONMENT', 'Environment update event for deployment completion', environmentDeploymentMetadata.generateEnvironmentMap("deployment_completed", environment_logical_id, null), environment_logical_id)
        events.sendCompletedEvent('UPDATE_DEPLOYMENT', "Deployment completion Event for ${env_key} deployment", environmentDeploymentMetadata.generateDeploymentMap("successful", environment_logical_id, gitCommitHash), environment_logical_id)
        events.sendCompletedEvent('DEPLOY_TO_AWS', 'Successfully deployed services to AWS', serviceContext, environment_logical_id)

      } //end of withCredentials
    } //end of deployment to an environment
  }
}

def handleS3BucketError(deployOutput, s3BucketName, environment_logical_id, envBucketKey, isEventSchdld) {
  def s3BucketErrString = "CloudFormation - CREATE_FAILED - AWS::S3::Bucket - "
  s3BucketName = lambdaEvents.getEventResourceNamePerEnvironment(s3BucketName, environment_logical_id, "-")
  if (deployOutput.contains(s3BucketErrString)) {
    if (lambdaEvents.checkS3BucketExists(s3BucketName)) {
      isLambdaUpdateRequired = true
      lambdaEvents.removeS3EventsFromServerless(isEventSchdld)
      deleteAndRedeployService(environment_logical_id, envBucketKey)
    } else {
      error "Error occured while accessing the s3 bucket"
    }
  } else {
    echo "Exception occured while serverless deployment to ${environment_logical_id} environment : $deployOutput"
    error "Exception occured while serverless deployment to ${environment_logical_id} environment"
  }
}

def handleSqsError(deployOutput, service_config, environment_logical_id, envBucketKey) {
  def event_source_sqs_arn = service_config['event_source_sqs']
  def sqsErrString = "CloudFormation - CREATE_FAILED - AWS::SQS::Queue - "
  if (deployOutput.contains(sqsErrString)) {
    def event_source_sqs = lambdaEvents.getSqsQueueName(event_source_sqs_arn, environment_logical_id)
    event_source_sqs_arn = lambdaEvents.getEventResourceNamePerEnvironment(event_source_sqs_arn, environment_logical_id, "_")
    def lambda_arn = "arn:aws:lambda:${configLoader.AWS.REGION}:${configLoader.AWS.ACCOUNTID}:function:${configLoader.INSTANCE_PREFIX}-${service_config['domain']}-${service_config['service']}-${environment_logical_id}"
    if (lambdaEvents.checkSqsQueueExists(event_source_sqs)) {
      if (lambdaEvents.checkIfDifferentFunctionTriggerAttached(event_source_sqs_arn, lambda_arn)) {
        error "Queue contains a different function trigger already. Please remove the existing function trigger and try again."
      }
      lambdaEvents.updateSqsResourceServerless()
      deleteAndRedeployService(environment_logical_id, envBucketKey)
    } else {
      error "Error occured while accessing the SQS queue"
    }
  } else {
    echo "Exception occured while serverless deployment to ${environment_logical_id} environment : $deployOutput"
    error "Exception occured while serverless deployment to ${environment_logical_id} environment"
  }
}

def handleKinesisStreamError(deployOutput, event_source_kinesis_stream_arn, environment_logical_id, envBucketKey) {
  def kinesisStreamErrString = "CloudFormation - CREATE_FAILED - AWS::Kinesis::Stream - "
  def event_source_kinesis = lambdaEvents.splitAndGetResourceName(event_source_kinesis_stream_arn, environment_logical_id)
  event_source_kinesis_stream_arn = lambdaEvents.getEventResourceNamePerEnvironment(event_source_kinesis_stream_arn, environment_logical_id, "_")
  def lambda_arn = "arn:aws:lambda:${configLoader.AWS.REGION}:${configLoader.AWS.ACCOUNTID}:function:${configLoader.INSTANCE_PREFIX}-${service_config['domain']}-${service_config['service']}-${environment_logical_id}"
  if (deployOutput.contains(kinesisStreamErrString)) {
    if (lambdaEvents.checkKinesisStreamExists(event_source_kinesis)) {
      if (lambdaEvents.checkIfDifferentFunctionTriggerAttached(event_source_kinesis_stream_arn, lambda_arn)) {
        error "Kinesis stream contains a different function trigger already. Please remove the existing function trigger and try again."
      }
      lambdaEvents.updateKinesisResourceServerless(event_source_kinesis_stream_arn)
      deleteAndRedeployService(environment_logical_id, envBucketKey)
    } else {
      error "Error occured while accessing the Kinesis stream"
    }
  } else {
    echo "Exception occured while serverless deployment to ${environment_logical_id} environment : $deployOutput"
    error "Exception occured while serverless deployment to ${environment_logical_id} environment"
  }
}

def handleDynamoDbStreamError(deployOutput, service_config, environment_logical_id, envBucketKey) {
  def dynamodbTableErrString = "CloudFormation - CREATE_FAILED - AWS::DynamoDB::Table - "
  def event_source_dynamodb_table_arn = service_config['event_source_dynamodb']
  def event_source_dynamodb = lambdaEvents.splitAndGetResourceName(event_source_dynamodb_table_arn, environment_logical_id)
  event_source_dynamodb_table_arn = lambdaEvents.getEventResourceNamePerEnvironment(event_source_dynamodb_table_arn, environment_logical_id, "_")
  def lambda_arn = "arn:aws:lambda:${configLoader.AWS.REGION}:${configLoader.AWS.ACCOUNTID}:function:${configLoader.INSTANCE_PREFIX}-${service_config['domain']}-${service_config['service']}-${environment_logical_id}"
  if (deployOutput.contains(dynamodbTableErrString)) {
    if (lambdaEvents.checkDynamoDbTableExists(event_source_dynamodb)) {
      def stream_details = lambdaEvents.getDynamoDbStreamDetails(event_source_dynamodb)
      if (!stream_details.isNewStream) {
        if (lambdaEvents.checkIfDifferentFunctionTriggerAttached(stream_details.StreamArn, lambda_arn)) {
          error "Dynamodb stream contains a different function trigger already. Please remove the existing function trigger and try again."
        }
      } else {
        events.sendCompletedEvent('CREATE_ASSET', null, utilModule.generateAssetMap("aws", stream_details.StreamArn, "dynamodb_stream", service_config), environment_logical_id);
      }
      lambdaEvents.updateDynamoDbResourceServerless(stream_details.StreamArn)
      deleteAndRedeployService(environment_logical_id, envBucketKey)
    } else {
      error "Error occured while accessing the dynamodb stream"
    }
  } else {
    echo "Exception occured while serverless deployment to ${environment_logical_id} environment : $deployOutput"
    error "Exception occured while serverless deployment to ${environment_logical_id} environment"
  }
}

def deleteAndRedeployService(environment_logical_id, envBucketKey){
  echoServerlessFile()
  sh "serverless remove --stage $environment_logical_id -v --bucket ${configLoader.AWS.S3[envBucketKey]}"
  def redeployOutput = servelessDeploy(environment_logical_id, envBucketKey)
  if (redeployOutput != 'success') {
    echo "Exception occured while serverless deployment to ${environment_logical_id} environment : $redeployOutput"
    error "Exception occured while serverless deployment to ${environment_logical_id} environment"
  }
}

def echoServerlessFile() {
  def serverlessyml = readFile('serverless.yml').trim()
  echo "serverless file data $serverlessyml"
}

def servelessDeploy(env, envBucketKey){
  try {
    sh "serverless deploy --stage ${env} -v --bucket ${configLoader.AWS.S3[envBucketKey]} > output.log"
    return "success"
  } catch (ex) {
    echo "Serverless deployment failed due to $ex"
  }

  def outputLog = readFile('output.log').trim()
  echo "serverless deployment log $outputLog"
  return outputLog
}

def addEvents(def isScheduleEnabled, def isEc2EventEnabled, def isS3EventEnabled, def isSQSEventEnabled, def isStreamEnabled, def isDynamoDbEnabled) {
  echo "addEvents to serverless.yml file"
  def sedCommand = "s/eventsDisabled/events/g";
  if (!isScheduleEnabled) {
    sedCommand = sedCommand + "; /#Start:isScheduleEnabled/,/#End:isScheduleEnabled/d"
  }
  if (!isEc2EventEnabled) {
    sedCommand = sedCommand + "; /#Start:isEc2EventEnabled/,/#End:isEc2EventEnabled/d"
  }
  if (!isS3EventEnabled) {
    sedCommand = sedCommand + "; /#Start:isS3EventEnabled/,/#End:isS3EventEnabled/d"
  }
  if (!isSQSEventEnabled) {
    sedCommand = sedCommand + "; /#Start:isSQSEventEnabled/,/#End:isSQSEventEnabled/d"
  }
  if (!isStreamEnabled) {
    sedCommand = sedCommand + "; /#Start:isStreamEnabled/,/#End:isStreamEnabled/d"
  }
  if (!isDynamoDbEnabled) {
    sedCommand = sedCommand + "; /#Start:isDynamoDbEnabled/,/#End:isDynamoDbEnabled/d"
  }

  sh "sed -i -- '$sedCommand' ./serverless.yml"
  echoServerlessFile()
  echo "------------------------DONE--------------"
}

def removeEventResources(){
  sh "sed -i -- '/#Start:resources/,/#End:resources/d' ./serverless.yml"
}


/**
 */
def isEnabled(config, key) {
  if (config.containsKey(key)) {
    return true
  } else {
    return false
  }
}

def LoadConfiguration() {
  def result = readFile('deployment-env.yml').trim()
  echo "result of yaml parsing....$result"
  def prop = [:]
  def resultList = result.tokenize("\n")

  // delete commented lines
  def cleanedList = []
  for (i in resultList) {
    if (i.toLowerCase().startsWith("#")) { } else {
      cleanedList.add(i)
    }
  }
  // echo "result of yaml parsing after clean up....$cleanedList"
  for (item in cleanedList) {
    // Clean up to avoid issues with more ":" in the values
    item = item.replaceAll(" ", "").replaceFirst(":", "#");
    def eachItemList = item.tokenize("#")
    //handle empty values
    def value = null;
    if (eachItemList[1]) {
      value = eachItemList[1].trim();
    }

    if (eachItemList[0]) {
      prop.put(eachItemList[0].trim(), value)
    }

  }
  echo "Loaded configurations....$prop"
  return prop
}


/** Run validation based on runtime
 */
/*def runValidation(String runtime) {


} */

/**	Build project based on runtime
 */
def buildLambda(String runtime) {
  echo "installing dependencies for $runtime"
  if (runtime.indexOf("nodejs") > -1) {
    sh "npm install --save"
  } else if (runtime.indexOf("java") > -1) {
    sh "mvn package"
  } else if (runtime.indexOf("python") > -1) {
    // install requirements.txt in library folder, these python modules will be a part of deployment package
    sh "rm -rf library"
    sh "mkdir library"
    sh "pip install -r requirements.txt -t library"
    sh "touch library/__init__.py"
    // create virtual environment and install pytest
    sh """
		pip install virtualenv
		virtualenv venv
		. venv/bin/activate
		pip install pytest
		"""
  }
}

/** Reset credentials
 */
def resetCredentials() {
  echo "resetting AWS credentials"
  sh "aws configure set profile.tmoDevOps.aws_access_key_id XXXXXXXXXXXXXXXXXXXXXXXXXX"
  sh "aws configure set profile.tmoDevOps.aws_secret_access_key XXXXXXXXXXXXXXXXXXXXXX"
}

/** Validate basic configurations in the deployment yaml file and error if any keys are
	missing.
*/
def validateDeploymentConfigurations(def prop) {
	if (prop.containsKey("service")) {
		if (prop['service'] == "") {
			error "Wrong configuration. Value for Key 'service' is missing in the configuration"
		}
	} else {
		error "Wrong configuration. Key 'service' is missing in the configuration"
	}
	if (prop.containsKey("providerRuntime")) {
		def _runtime = prop['providerRuntime']
		if (_runtime == "") {
			error "Wrong configuration. Value for Key 'providerRuntime' is missing in the configuration"
		} else {
			def validRuntimes = ["nodejs4.3", "nodejs6.10", "nodejs8.10", "python2.7", "java8"] //@TODO. Add more runtime supports.
			def flag = false

			for (int i = 0; i < validRuntimes.size(); i++) {
				if (_runtime == validRuntimes[i]) {
					flag = true
				}
			}

			if (!flag) {
				echo "$flag"
				error "Runtime given in the configuration is not valid."
			}
		}
	} else {
		error "Wrong configuration. Key 'providerRuntime' is missing in the configuration"
	}
	if (prop.containsKey("providerTimeout")) {
		if (prop['providerTimeout'] == "") {
			error "Wrong configuration. Value for Key 'providerTimeout' is missing in the configuration"
		} else if (Integer.parseInt(prop['providerTimeout']) > 300) { // Should not be a high
			error "Wrong configuration. Value for Key 'providerTimeout' should be a less than 160"
		}
	} else {
		error "Wrong configuration. Key 'providerTimeout' is missing in the configuration"
	}

	if (prop.containsKey("region")) {
		if (prop['region'] == "") {
			error "Wrong configuration. Value for Key 'region' is missing in the configuration"
		} else if (prop['region'] != configLoader.AWS.REGION) {
			error "Wrong configuration. Value for Key 'region' should be ${region}"
		}
	} else {
		error "Wrong configuration. Key 'region' is missing in the configuration"
	}

	def runtime = prop['providerRuntime']
	if (runtime.indexOf("java") > -1) {

		if (prop.containsKey("artifact")) {
			if (prop['artifact'] == "") {
				error "Wrong configuration. Value for Key 'artifact' is missing in the configuration"
			}
		} else {
			error "Wrong configuration. Key 'artifact' is missing in the configuration"
		}

		if (prop.containsKey("mainClass")) {
			if (prop['mainClass'] == "") {
				error "Wrong configuration. Value for Key 'mainClass' is missing in the configuration"
			}
		} else {
			error "Wrong configuration. Key 'mainClass' is missing in the configuration"
		}
	}
}


def loadServerlessConfig(String runtime, def isEventSchdld, def isScheduleEnabled, def isEc2EventEnabled, def isS3EventEnabled, def isSQSEventEnabled, def isStreamEnabled, def isDynamoDbEnabled) {

  def configPackURL = scmModule.getCoreRepoCloneUrl("serverless-config-pack")

  dir('_config') {
    checkout([$class: 'GitSCM', branches: [
      [name: '*/master']
    ], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
        [credentialsId: repo_credential_id, url: configPackURL]
      ]])
  }

  echo "isEventSchdld: $isEventSchdld,isScheduleEnabled: $isScheduleEnabled, isEc2EventEnabled: $isEc2EventEnabled,  isS3EventEnabled: $isS3EventEnabled, isStreamEnabled: $isStreamEnabled, isDynamoDbEnabled: $isDynamoDbEnabled"

  if (runtime.indexOf("nodejs") > -1) {
    sh "cp _config/serverless-nodejs.yml ./serverless.yml"
  } else if (runtime.indexOf("java") > -1) {
    sh "cp _config/serverless-java.yml ./serverless.yml"
  } else if (runtime.indexOf("python") > -1) {
    sh "cp _config/serverless-python.yml ./serverless.yml"
  }

  if (isDynamoDbEnabled || isSQSEventEnabled) {
    sh "cp _config/aws-events-policies/custom-policy.yml ./policyFile.yml"
  }

  if (isEventSchdld || isS3EventEnabled) {
    addEvents(isScheduleEnabled, isEc2EventEnabled, isS3EventEnabled, isSQSEventEnabled, isStreamEnabled, isDynamoDbEnabled)
  }

  if (!isEventSchdld && !isS3EventEnabled) {
    removeEventResources()
  }

  echoServerlessFile()
}

def echoServiceInfo(String env) {
  try {
    echo "Deployment info:"
    sh "serverless --stage $env info -v > deploy-info.txt"

    def arn = "unknown"
    def svc_response = "unknown"
    def result = readFile('deploy-info.txt').trim()
    def resultList = result.tokenize("\n")

    for (item in resultList) {
      if (item.startsWith("HandlerLambdaFunctionQualifiedArn")) {
        arn = item.trim().substring(35)
        version = arn.tokenize(':').last()
        arn = arn.substring(0, arn.length() - version.length() - 1)

        svc_response = "Your service endpoint: " + arn
      }
    }

    echo "==============================================================================================="
    echo svc_response
    echo "==============================================================================================="
  } catch (Exception ex) {
    echo " Error while getting service info : " + ex.getMessage()
  }
}

/**
	Create the subscription filters and loggroup if not existing
**/
def createSubscriptionFilters(stackName, region, roleId) {
  def lambda = "/aws/lambda/${stackName}"
  def logStreamer = configLoader.AWS.KINESIS_LOGS_STREAM.PROD

  try {
    sh "aws logs create-log-group --log-group-name ${lambda} --region ${region}"
  } catch (Exception ex) { } // ignore if already existing

  try {
    filter_json = sh(
      script: "aws logs describe-subscription-filters --output json --log-group-name \"${lambda}\" --region ${region}",
      returnStdout: true
    ).trim()
    echo "${filter_json}"
    def resultJson = parseJson(filter_json)
    filtername = resultJson.subscriptionFilters[0].filterName
    echo "removing existing filter... $filtername"
    if (filtername != "" && !filtername.equals(lambda)) {
      sh "aws logs delete-subscription-filter --output json --log-group-name \"${lambda}\" --filter-name \"${filtername}\" --region ${region}"
    }
  } catch (Exception ex) { } // ignore error if not created yet
  try {
    sh "aws logs put-subscription-filter --output json --log-group-name \"${lambda}\" --filter-name \"${lambda}\" --filter-pattern \"\" --destination-arn \"${logStreamer}\" --region ${region} --role-arn \"arn:aws:iam::${roleId}:role/${configLoader.INSTANCE_PREFIX}_basic_execution\""
  } catch (Exception ex) {
    echo "error occured: " + ex.getMessage()
  }
}


def writeServerlessFile(config, env) {
  sh "pwd"
  sh "sed -i -- 's/\${file(deployment-env.yml):service}/${configLoader.INSTANCE_PREFIX}-${config['domain']}-${config['service']}/g' serverless.yml"
  sh "sed -i -- 's/{inst_stack_prefix}/${configLoader.INSTANCE_PREFIX}/g' serverless.yml"
  sh "sed -i -- 's/{env-stage}/${env}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):region}/${config['region']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):domain, self:provider.domain}/${config['domain']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):owner, self:provider.owner}/${config['created_by']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):providerRuntime}/${config['providerRuntime']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):providerMemorySize}/${config['providerMemorySize']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):providerTimeout}/${config['providerTimeout']}/g' serverless.yml"
  sh "sed -i -- 's|\${file(deployment-env.yml):eventScheduleRate}|${config['eventScheduleRate']}|g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):eventScheduleEnable}/${config['eventScheduleEnable']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):securityGroupIds}/${config['securityGroupIds']}/g' serverless.yml"
  sh "sed -i -- 's/\${file(deployment-env.yml):subnetIds}/${config['subnetIds'] }/g' serverless.yml"

  if (config['artifact']) {
    sh "sed -i -- 's/\${file(deployment-env.yml):artifact}/${config['artifact']}/g' serverless.yml"
  }

  if (config['mainClass']) {
    sh "sed -i -- 's/\${file(deployment-env.yml):mainClass}/${config['mainClass']}/g' serverless.yml"
  }

  if (config['event_source_s3']) {
    def event_source_s3 = lambdaEvents.getEventResourceNamePerEnvironment(config['event_source_s3'], env, "-")
    sh "sed -i -- 's/{event_source_s3}/${event_source_s3}/g' ./serverless.yml"
    sh "sed -i -- 's/{event_action_s3}/${config['event_action_s3']}/g' ./serverless.yml"
  }

  if (config['event_source_sqs']) {
    def event_source_sqs = lambdaEvents.getSqsQueueName(config['event_source_sqs'], env)
    def event_sqs_arn = lambdaEvents.getEventResourceNamePerEnvironment(config['event_source_sqs'], env, "_")
    sh "sed -i -- 's/{event_source_sqs}/${event_source_sqs}/g' ./serverless.yml"
    sh "sed -i -- 's|{event_sqs_arn}|${event_sqs_arn}|g' ./serverless.yml"
    sh "sed -i -- 's|{event_sqs_arn}|${event_sqs_arn}|g' policyFile.yml"
    sh "sed -i -- 's/resourcesDisabled/resources/g' ./serverless.yml"
  }

  if (config['event_source_kinesis']) {
    def event_source_kinesis = lambdaEvents.splitAndGetResourceName(config['event_source_kinesis'], env)
    sh "sed -i -- 's/resourcesDisabled/resources/g' ./serverless.yml"
    sh "sed -i -- 's|{event_source_kinesis}|${event_source_kinesis}|g' ./serverless.yml"
  }

  if (config['event_source_dynamodb']) {
    def event_source_dynamodb = lambdaEvents.splitAndGetResourceName(config['event_source_dynamodb'], env)
    sh "sed -i -- 's/resourcesDisabled/resources/g' ./serverless.yml"
    sh "sed -i -- 's|{event_source_dynamodb}|${event_source_dynamodb}|g' serverless.yml"
  }

  if (config['event_source_dynamodb'] || config['event_source_sqs']) {
    sh "sed -i -e 's|\${file(deployment-env.yml):iamRoleARN}|customEventRole|g' serverless.yml"
  } else {
    sh "sed -i -e 's|\${file(deployment-env.yml):iamRoleARN}|${config['iamRoleARN']}|g' serverless.yml"
  }

}

/**
 * For getting token to access catalog APIs.
 * Must be a service account which has access to all services
 */
def setCredentials() {
  def loginUrl = g_base_url + '/jazz/login'
  def token

  withCredentials([
    [$class: 'UsernamePasswordMultiBinding', credentialsId: g_svc_admin_cred_ID, passwordVariable: 'PWD', usernameVariable: 'UNAME']
  ]) {
    echo "user name is $UNAME"

    def login_json = []

    login_json = [
      'username': UNAME,
      'password': PWD
    ]
    def tokenJson_token = null
    def payload = JsonOutput.toJson(login_json)

    try {
      token = sh(script: "curl --silent -X POST -k -v \
				-H \"Content-Type: application/json\" \
					$loginUrl \
				-d \'${payload}\'", returnStdout: true).trim()

      def tokenJson = parseJson(token)
      tokenJson_token = tokenJson.data.token

      return tokenJson_token
    } catch (e) {
      echo "error occured: " + e.getMessage()
      error "error occured: " + e.getMessage()
    }
  }
}

/**
 * Send email to the recipient with the build status and any additional text content
 * Supported build status values = STARTED, FAILED & COMPLETED
 * @return
 */
def send_status_email(config, build_status, email_content) {
  if (domain != "jazz") {
    echo "Sending build notification to ${config['created_by']}"
    def body_subject = ''
    def body_text = ''
    def cc_email = ''
    def body_html = ''
    if (build_status == 'STARTED') {
      echo "email status started"
      body_subject = 'Jazz Build Notification: Deployment STARTED for service: ' + config['service']
    } else if (build_status == 'FAILED') {
      echo "email status failed"
      def build_url = env.BUILD_URL + 'console'
      body_subject = 'Jazz Build Notification: Deployment FAILED for service: ' + config['service']
      body_text = body_text + '\n\nFor more details, please click this link: ' + build_url
    } else if (build_status == 'COMPLETED') {
      body_subject = 'Jazz Build Notification: Deployment COMPLETED successfully for service: ' + config['service']
    } else {
      echo "Unsupported build status, nothing to email.."
      return
    }
    if (email_content != '') {
      body_text = body_text + '\n\n' + email_content
    }
    def fromStr = 'Jazz Admin <' + configLoader.JAZZ.ADMIN + '>'
    body = JsonOutput.toJson([
      from: fromStr,
      to: config['created_by'],
      subject: body_subject,
      text: body_text,
      cc: cc_email,
      html: body_html
    ])

    try {
      def sendMail = sh(script: "curl -X POST \
					${g_base_url}/jazz/email \
					-k -v -H \"Authorization: $auth_token\" \
					-H \"Content-Type: application/json\" \
					-d \'${body}\'", returnStdout: true).trim()
      def responseJSON = parseJson(sendMail)
      if (responseJSON.data) {
        echo "successfully sent e-mail to ${config['created_by']}"
      } else {
        echo "exception occured while sending e-mail: $responseJSON"
      }
    } catch (e) {
      echo "Failed while sending build status notification"
    }
  }
}


/**
 *  Function to get arns of the triggers/events configured for the Lambda.
 *
 */
def getEventsArn(config, env) {
  def eventsArn = []
  try {
    def lambdaFnName = "${configLoader.INSTANCE_PREFIX}-${config['domain']}-${config['service']}-${env}"
    def lambdaPolicyTxt = sh(script: "aws lambda get-policy --region ${configLoader.AWS.REGION} --function-name $lambdaFnName --output json", returnStdout: true)

    def policyLists = null
    if (lambdaPolicyTxt) {
      def lambdaPolicyJson = new groovy.json.JsonSlurperClassic()
      policyLists = lambdaPolicyJson.parseText(lambdaPolicyJson.parseText(lambdaPolicyTxt).Policy)
      if (policyLists) {
        for (st in policyLists.Statement) {
          if (st.Principal.Service == "events.amazonaws.com") {
            if (st.Condition.ArnLike["AWS:SourceArn"]) {
              eventsArn.push(st.Condition.ArnLike["AWS:SourceArn"])
            }
          }
        }
      }
    }
    return eventsArn
  } catch (ex) {
    // Skip the 'ResourceNotFoundException' when deploying first time. Workflow can't fail here.
    echo "Can't fetch the events policy configurations for lambda. " + ex.getMessage()
    return []
  }
}

def getLambdaARN(String stackName) {
  def ARN = "";

  try {
    def cloudformation_resources = "";
    cloudformation_resources = sh(returnStdout: true, script: "aws cloudformation describe-stacks --output json --stack-name ${stackName} --profile cloud-api")

    def parsedObject = parseJson(cloudformation_resources);
    def outputs = parsedObject.Stacks[0].Outputs;

    for (output in outputs) {
      if (output.OutputKey == "HandlerLambdaFunctionQualifiedArn") {
        ARN = output.OutputValue
      }
    }
  } catch (ex) {
    error ex.getMessage();
  }

  return ARN;
}

/** Run validation based on runtime
 */
def runValidation(String runtime) {
  if (runtime.indexOf("nodejs") > -1) {
    echo "running validations for $runtime"
    sh "jshint *.js"
  } else if (runtime.indexOf("java") > -1) {
    echo "running validations for $runtime"
    sh "java -cp ${configLoader.CODE_QUALITY.SONAR.CHECKSTYLE_LIB} com.puppycrawl.tools.checkstyle.Main -c sun_checks.xml src"
  } else if (runtime.indexOf("python") > -1) {
    echo "running validations for $runtime"
  }
}

@NonCPS
def parseJson(jsonString) {
  def lazyMap = new groovy.json.JsonSlurperClassic().parseText(jsonString)
  def m = [:]
  m.putAll(lazyMap)
  return m
}

def updatePlatformServiceTemplate(config, roleARN, roleId, current_environment, repo_name) {
  echo "loadServerlessConfig......."
  def jenkinsURL = JenkinsLocationConfiguration.get().getUrl().replaceAll("/", "\\\\/")
  serviceConfigdata.initialize(configLoader, roleARN, config['region'], roleId, jenkinsURL, current_environment, repo_name, utilModule)
  serviceConfigdata.loadServiceConfigurationData()
}
/*
 * Load build modules
 */
def loadBuildModules(buildModuleUrl) {
  dir('build_modules') {
    checkout([$class: 'GitSCM', branches: [
      [name: '*/master']
    ], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [
        [credentialsId: repo_credential_id, url: buildModuleUrl]
      ]])

    def resultJsonString = readFile("jazz-installer-vars.json")
    configModule = load "config-loader.groovy"
    configLoader = configModule.initialize(resultJsonString)
    echo "config loader loaded successfully."

    scmModule = load "scm-module.groovy"
    scmModule.initialize(configLoader)
    echo "SCM module loaded successfully."

    serviceConfigdata = load "service-configuration-data-loader.groovy"
    echo "Service configuration module loaded successfully."

    events = load "events-module.groovy"
    echo "Event module loaded successfully."

    serviceMetadataLoader = load "service-metadata-loader.groovy"
    serviceMetadataLoader.initialize(configLoader)
    echo "Service metadata loader module loaded successfully."

    utilModule = load "utility-loader.groovy"
    echo "Util module loaded successfully."

    lambdaEvents = load "aws-lambda-events-module.groovy"
    lambdaEvents.initialize(configLoader, utilModule)
    echo "Lambda event module loaded successfully."

    sonarModule = load "sonar-module.groovy"
    echo "Sonar module loaded successfully."

    environmentDeploymentMetadata = load "environment-deployment-metadata-loader.groovy"
    echo "Environment deployment data loader module loaded successfully."
  }
}

def getBuildModuleUrl() {
  if (scm_type && scm_type != "bitbucket") {
    // right now only bitbucket has this additional tag scm in its git clone path
    return "http://${repo_base}/${repo_core}/jazz-build-module.git"
  } else {
    return "http://${repo_base}/scm/${repo_core}/jazz-build-module.git"
  }
}

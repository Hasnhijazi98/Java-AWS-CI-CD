# Java-AWS-CI-CD



***********************************************************************
****************JAVA APP DEPLOY TO AWS*********************************
***********************************************************************

so we forked a existing gitlab repo and opened it in our intelliJ, don't forget to create a anew access token for the project, and watch out for the protected branches, by
default the forked project pranches are protected, now we'll add a simple .gitlab-ci.yml, here is it :

stages:
  - build

build:
  stage: build
  image: openjdk:12-alpine
  script:
    - ./gradlew build
  artifacts:
    paths:
      - ./build/libs

a concept of pipelines files, is to do a "smoke test", it is a simple test just to check if our app is working checking a simple thing, so we add the smoke test to the code 
(code updated above)

NOW WE'RE GONNA DEPLOY TO AWS TWICE, ONCE MANUALLY USING ELASTIC BEANSTALK (THAT WORKS ONLY WOTH JAVA,NODE OR PHP APPS) and ONCE USING GITLAB (AND MAYBE ONE MORE USING AWS CLI)

DUDE !!! that guy on the video tutorial for deploying java jar app on EB aws server is missing a LOT of points !!! its' beccause of the last update made in 1st october 2024

However i got you, first of all, create a IAM, this is where you'll create a user, and you'll create a role(if it doesn't exists) called aws-elasticbeanstalk-ec2-role and give it the correct policies AWSElasticBeanstalkWebTier or AWSElasticBeanstalkFullAccess

Then, create a EC2 instance, and don't forget choose the free bundle settings while doing that :
	navigate to EC2 abnd unjder Instances, create launch template
	Name: Give it a name, like MyElasticBeanstalkTemplate.
	AMI ID: Choose the Amazon Machine Image (AMI) that matches the environment you want. For example, if you are using Amazon Linux 2, select that.
	Instance type: Select the appropriate instance type (like t2.micro for testing or development environments).
	IAM Role: Attach the necessary IAM role (like the one just created in previous step) to give instances the required permissions
	
and then normally you must create a elastic beanstalk environment, make sure in othe second step to put the iam role you created, and you must also associate
the launch template you created  to the EB environment you created, to do so, normally you can do it in the environments --> configurations, if you don't have it,
add it manually in the code, that's what i did, in the root of the gradle java project, i added this file, where i enable the auto scaling option, and determine the 
launch template i created :

	Resources:
	  AWSEBAutoScalingGroup:
	    Type: AWS::AutoScaling::AutoScalingGroup
	    Properties:
	      LaunchTemplate:
		LaunchTemplateName: MyElasticBeanstalkTemplate
		Version: '$Latest'
		
now now, let's go wwith the deployment from gitlab CI/CD :
	-create a bucket on aws : - in AMZAON S3 --> create Bucket, just leave everything as they are by default and create it
	-now you'll see you already have another bucket that was created for the firt deployment on elastic beanstalk so don't panic
	-now you can upload manually your jar, but we want to upload it vie our gitlab
	
now in gitlab : 
	- copy the name of the bucket, and paste it in a new environment variable in your project in gitlab, let's mae the variable S3_VAR
	- in your  IAM console on aws, get the key acess id and the secret access key (you cannot reveal your secret acess key, if you didn't already saved it at the user creation, then create another user), and then add the 2 values in gitlab environment variables and name them AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
	- add the S3FullAccess permission to the user
	- and then update your gitlab-ci.yml code, add a deploy stage and give it the corresponding elements:
		. image : amazon cli
		. script : copy the jar file to the S3 bucket, using the S3_INSTANCE environment variable so it detects which bucket we want to add the jar in it
		
now we deployed succesfully on a S3 bucket, now we want to associate it to a elastic beanstalk environment,to do so:
	- give the IAM user permission for the elasticbeanstalk full access
	- add to the dploy stage the script to deploy to the aws elastic beanstalk existing environment and create the app, and don't forget to versionize the app
	- you're gonna see a lot of vars now, it's because we want to control the versionning of the application using the IID Pipeline

updated yml:

variables:
  ARTIFACT_NAME: cars-api-v$CI_PIPELINE_IID.jar
  APP_NAME: cars-api

stages:
  - build
  - test
  - deploy

image: openjdk:12-alpine

build:
  stage: build
  script:
    - ./gradlew build
    - mv ./build/libs/cars-api.jar ./build/libs/$ARTIFACT_NAME
  artifacts:
    paths:
      - ./build/libs

smoke test:
  stage: test
  script:
    - apk --no-cache add curl
    - java -jar ./build/libs/$ARTIFACT_NAME &
    - sleep 30
    - curl http://localhost:5000/statistics/age | grep "3"

aws deploy :
  stage: deploy
  image:
    name: amazon/aws-cli
    entrypoint: [""]
  script:
    - aws configure set region us-east-1
    - aws s3 cp ./build/libs/cars-api.jar s3://$S3_INSTANCE/cars-api.jar || { echo "Upload failed"; exit 1; }
    - aws elasticbeanstalk create-application-version --application-name $APP_NAME --version-label $CI_PIPELINE_IID --source-bundle S3Bucket=$S3_INSTANCE,S3Key=$ARTIFACT_NAME
    - aws elasticbeanstalk update-environment --application-name $APP_NAME --environment-name "production" --version-label=$CI_PIPELINE_IID
    
NOW, as last touch, let's add some unit tests, well we already have in the project, but we want to run them in the pipeline, for that, we add the test job where we run the tests and save up in artifacts the reports that are under build/reports and build/test-results
test job:

TU:
  stage: test
  script:
    - ./gradlew test
  artifacts:
    when: always
    paths:
      - build/reports/tests/test
    reports:
      junit: build/test-results/test
      
   we added it in teh testing stage, so we'll have in the testing stage 2 jobs in parallel (TU and smoke test) and we add the reports-junit so we can have a report and result tab in the pipeline on gitlab
   in gitlab pipeline, in the artifacts of the job, check the index.html file, it'll have the success rate of the tests and other détails
   
   
   SO CONGRATS !!! here we finished this training, some few recommandation:
   	try to add a code quality testing stage
   	try to add a post deploy stage where you test your apis, you can generate api tests from postman
   	put a threshold for code quality and test stage, if the success rate is below these thresholds, the deploy fails

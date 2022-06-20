Exercise: Promote to Production
Write a set of jobs that promotes a new environment to production and decommissions the old environment in an automated process.

Prerequisites
You should have finished the previous:

Exercise: Remote Control Using Ansible,
Exercise: Infrastructure Creation,
Exercise: Config and Deployment, and
Exercise: Rollback
Instructions
There are a few manual steps we need to take to "prime the pump". These manual steps enable steps that will be automated later.

Step 1. Create an S3 bucket (Manually)
Create a public S3 Bucket (e.g., mybucket644752792305) manually. If you need help with this, follow the instructions found in the documentation.
Create a simple HTML page (version 1) and name it index.html. It could be as simple as:

<!DOCTYPE html>
<html>
  <head>
      <title>Version 1</title>
  </head>

  <body>
      <h1>Hello World - version 1</h1>
  </body>
</html>

Step 2. Create a Cloudformation Stack (Manually)
Use a Cloudformation template to create a Cloudfront Distribution, and later in this exercise, use the same template to modify the Cloudfront distribution.

Confused between Cloudformation (IaC) vs Cloudfront (CDN)? We encourage you to read the basic definition from the official docs.

Cloudformation is a service that allows creating any AWS resource using the YAML template files. In other words, it supports Infrastructure as a Code (IaC). Cloudfront is a network of servers that caches the content, such as website media, geographically close to consumers. Cloudfront is a Content Delivery Network (CDN) service.

Create a Cloudformation template file named cloudfront.yml with the following contents:
Parameters:
# Existing Bucket name
PipelineID:
  Description: Existing Bucket name
  Type: String
Resources:
    CloudFrontOriginAccessIdentity:
      Type: "AWS::CloudFront::CloudFrontOriginAccessIdentity"
      Properties:
        CloudFrontOriginAccessIdentityConfig:
          Comment: Origin Access Identity for Serverless Static Website
    WebpageCDN:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Origins:
            - DomainName: !Sub "${PipelineID}.s3.amazonaws.com"
              Id: webpage
              S3OriginConfig:
                OriginAccessIdentity: !Sub "origin-access-identity/cloudfront/${CloudFrontOriginAccessIdentity}"
          Enabled: True
          DefaultRootObject: index.html
          DefaultCacheBehavior:
            ForwardedValues:
              QueryString: False
            TargetOriginId: webpage
            ViewerProtocolPolicy: allow-all
  Outputs:
    PipelineID:
      Value: !Sub ${PipelineID}
      Export:
        Name: PipelineID


The file above is also available here and it creates a Cloudfront Distribution that will connect to the existing bucket.

https://github.com/udacity/nd9991-c3-hello-world-exercise-solution/blob/main/cloudfront.yml

Execute the cloudfront.yml template above with the following command:
aws cloudformation deploy \
--template-file cloudfront.yml \
--stack-name production-distro \
--parameter-overrides PipelineID="${S3_BUCKET_NAME}" \ # Name of the S3 bucket you created manually.
--tags project=udapeople &

In the command above, replace ${S3_BUCKET_NAME} with the actual name of your bucket (e.g., mybucket644752792305). Also, note down the stack name - production-distro for the future. You will need the cloudfront.yml file in the automation steps later again.

Step 3. Cloudformation template to create a new S3 bucket
Caveat: Copy-pasting the YAML code below will break the indentation! Fix the indentation hierarchy before you push your updates to your Github repo.

Create another Cloudformation template named bucket.yml that will create a new bucket and bucket policy. You can use the following template:
Parameters:
# New Bucket name
MyBucketName:
  Description: Existing Bucket name
  Type: String
Resources:
WebsiteBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: !Sub "${MyBucketName}"
    AccessControl: PublicRead
    WebsiteConfiguration:
      IndexDocument: index.html
      ErrorDocument: error.html
WebsiteBucketPolicy:
  Type: AWS::S3::BucketPolicy
  Properties:
    Bucket: !Ref 'WebsiteBucket'
    PolicyDocument:
      Statement:
      - Sid: PublicReadForGetBucketObjects
        Effect: Allow
        Principal: '*'
        Action: s3:GetObject
        Resource: !Join ['', ['arn:aws:s3:::', !Ref 'WebsiteBucket', /*]]


https://github.com/udacity/nd9991-c3-hello-world-exercise-solution/blob/main/bucket.yml


Step 4. Update the CircleCI Config file
Job - Write a job named create_and_deploy_front_end that executes the bucket.yml template to:

Create a new S3 bucket and
Copy the contents of the current repo (production files) to the new bucket.

Your job should look like this:

# Executes the bucket.yml - Deploy an S3 bucket, and interface with that bucket to synchronize the files between local and the bucket.
# Note that the `--parameter-overrides` let you specify a value that override parameter value in the bucket.yml template file.
create_and_deploy_front_end:
docker:
 - image: amazon/aws-cli
steps:
 - checkout
 - run:
     name: Execute bucket.yml - Create Cloudformation Stack
     command: |
       aws cloudformation deploy \
       --template-file bucket.yml \
       --stack-name stack-create-bucket-${CIRCLE_WORKFLOW_ID:0:7} \
       --parameter-overrides MyBucketName="mybucket-${CIRCLE_WORKFLOW_ID:0:7}"
 # Uncomment the step below if yoou wish to upload all contents of the current directory to the S3 bucket
  - run: aws s3 sync . s3://mybucket-${CIRCLE_WORKFLOW_ID:0:7} --delete


  Please be mindful of the indentation. The job is also avilable in this file.

Notice we are passing in the CIRCLE_WORKFLOW_ID in mybucket-${CIRCLE_WORKFLOW_ID:0:7} to help form the name of our new bucket. This helps us to reference the bucket later, in another job/command.

Once this job runs successfully, it will change the cache in the CDN to the new index.html, which is also present here.

Towards the end of this exercise, we need another job for deleting the S3 bucket created manually (cleaning up after promotion). For this purpose, you need to know which pipeline ID was responsible for creating the S3 bucket created manually (the last successful production release). We can query Cloudformation for the old pipeline ID information.
Job - Write a CircleCI job named get_last_deployment_id that performs the query and saves the id to a file that we can persist to the workspace. For convenience, here's the job that you can use:
# Fetch and save the pipeline ID (bucket ID) responsible for the last release.
get_last_deployment_id:
  docker:
    - image: amazon/aws-cli
  steps:
    - checkout
    - run: yum install -y tar gzip
    - run:
        name: Fetch and save the old pipeline ID (bucket name) responsible for the last release.
        command: |
          aws cloudformation \
          list-exports --query "Exports[?Name==\`PipelineID\`].Value" \
          --no-paginate --output text > ~/textfile.txt
    - persist_to_workspace:
        root: ~/
        paths: 
          - textfile.txt 

In the job above, we are saving the bucket ID to a file and persist the file to the workspace for other jobs to access.

Job - Write a job named promote_to_production that executes our cloudfront.yml CloudFormation template used in the manual steps. Here's the job you can use:
# Executes the cloudfront.yml template that will modify the existing CloudFront Distribution, change its target from the old bucket to the new bucket - `mybucket-${CIRCLE_WORKFLOW_ID:0:7}`. 
# Notice here we use the stack name `production-distro` which is the same name we used while deploying to the S3 bucket manually.
promote_to_production:
  docker:
    - image: amazon/aws-cli
  steps:
    - checkout
    - run:
        name: Execute cloudfront.yml
        command: |
          aws cloudformation deploy \
          --template-file cloudfront.yml \
          --stack-name production-distro \
          --parameter-overrides PipelineID="mybucket-${CIRCLE_WORKFLOW_ID:0:7}"
Notice here we use the stack name production-distro which is the same name we used in the throw-away Cloudformation template above.

Job - Write a Job Named clean_up_old_front_end that uses the pipeline ID to destroy the previous production version's S3 bucket and CloudFormation stack. To achieve this, you need to retrieve from the workspace the file where the previous Pipeline ID was stored. Once you have the Pipeline ID, use the following commands to clean up:
# Destroy the previous production version's S3 bucket and CloudFormation stack. 
clean_up_old_front_end:
  docker:
    - image: amazon/aws-cli
  steps:
    - checkout
    - run: yum install -y tar gzip
    - attach_workspace:
        at: ~/
    - run:
        name: Destroy the previous S3 bucket and CloudFormation stack. 
        # Use $OldBucketID environment variable or mybucket644752792305 below.
        # Similarly, you can create and use $OldStackID environment variable in place of production-distro 
        command: |
          export OldBucketID=$(cat ~/textfile.txt)
          aws s3 rm "s3://${OldBucketID}" --recursive
          
Workflow - Define a Workflow that puts these jobs in order. Comment out the jobs that are not part of this exercise. Try to keep the workflow minimal. Your workflow will look as:
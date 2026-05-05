---
title: "Implement a data mesh pattern in Amazon SageMaker Catalog without changing applications"
url: "https://aws.amazon.com/blogs/big-data/implement-a-data-mesh-pattern-with-amazon-sagemaker-catalog-without-making-changes-to-your-applications/"
date: "Mon, 23 Feb 2026 19:41:32 +0000"
author: "Paolo Romagnoli"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p>When creating a project in <a href="https://aws.amazon.com/sagemaker/unified-studio/" rel="noopener noreferrer" target="_blank">Amazon SageMaker Unified Studio</a>, users select a project profile to define resources and tools to be provisioned in the project. These are used by <a href="https://aws.amazon.com/sagemaker/catalog/" rel="noopener noreferrer" target="_blank">Amazon SageMaker Catalog</a> to implement a data mesh pattern. Some users don’t want to take advantage of resources provisioned along with the project for various reasons. For instance, they may want to avoid making changes to their existing applications and data products.</p> 
<p>This post shows you how to implement a data mesh pattern by using Amazon SageMaker Catalog while keeping your current data repositories and consumer applications unchanged.</p> 
<h2>Solution overview</h2> 
<p>In this post, you will simulate a scenario based on data producer and data consumer that exists before Amazon SageMaker Catalog adoption. For this purpose, you will use a sample dataset to simulate existing data and simulate an existing application using an <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function. You can apply the same solution to your real-life data and workloads.</p> 
<p>The following diagram illustrates the solution architecture’s key configurations. In this architecture, the <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) bucket and the <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a> in the producer account simulate the existing data repository. The Lambda function in the consumer account simulates the existing consumer application.</p> 
<p><img alt="AWS cross-account data sharing via SageMaker &amp; Lake Formation: Producer publishes to catalog, Consumer subscribes &amp; accesses data" class="alignnone wp-image-87941 size-full" height="816" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-1.png" width="1292" /></p> 
<p>Here is a description of the key configurations highlighted in the architecture:</p> 
<ol> 
 <li>As part of an <a href="https://aws.amazon.com/sagemaker/" rel="noopener noreferrer" target="_blank">Amazon SageMaker</a> domain, create a producer project (associated to a producer account) and a consumer project (associated to a consumer account). Among other resources, a project <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) role is created for each project in the associated account.</li> 
 <li>In the producer account, use <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a> to grant producer project’s IAM role permissions to access the existing data asset.</li> 
 <li>Publish the data asset in the Amazon SageMaker Catalog from the producer project.</li> 
 <li>Subscribe the data asset from the consumer project.</li> 
 <li>In the consumer account, configure your Lambda function to assume consumer project’s IAM role to access the subscribed data asset.</li> 
</ol> 
<p>The solution architecture is based on the following <a href="https://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services</a> (AWS) services and features:</p> 
<ul> 
 <li>Amazon SageMaker Catalog offers you a way to discover, govern, and collaborate on data and AI securely.</li> 
 <li>Amazon SageMaker Unified Studio provides a single data and AI development environment to discover and build with your data. Amazon SageMaker Unified Studio projects provide collaborative boundaries for users to accomplish data and AI tasks.</li> 
 <li><a href="https://aws.amazon.com/sagemaker/lakehouse/" rel="noopener noreferrer" target="_blank">The lakehouse architecture of Amazon SageMaker</a> is fully compatible with <a href="https://iceberg.apache.org/" rel="noopener noreferrer" target="_blank">Apache Iceberg</a>. It unifies data across Amazon S3 data lakes, <a href="https://aws.amazon.com/redshift/" rel="noopener noreferrer" target="_blank">Amazon Redshift</a> data warehouses, and third-party and federated data sources.</li> 
 <li>AWS Lake Formation, which you can use centrally to govern, secure, and share data for analytics and machine learning.</li> 
 <li>AWS Glue Data Catalog is a persistent metadata store for your data assets. It contains table definitions, job definitions, schemas, and other control information to help you manage your AWS Glue environment.</li> 
 <li>Amazon S3 is an object storage service that offers industry-leading scalability, data availability, security, and performance.</li> 
</ul> 
<h2>Setting up resources</h2> 
<p>In this section, you will prepare the resources and configurations you need for this solution.</p> 
<h3>Three AWS accounts</h3> 
<p>To follow this solution, you need three AWS accounts, and it’s better if they’re part of the same organization in <a href="https://aws.amazon.com/organizations/" rel="noopener noreferrer" target="_blank">AWS Organizations</a>:</p> 
<ul> 
 <li><strong>Producer account</strong> – Hosts the data asset to be published</li> 
 <li><strong>Consumer account</strong> – Hosts the application that consumes the data published from the producer account</li> 
 <li><strong>Governance account</strong> – Where the Amazon SageMaker Unified Studio domain is configured</li> 
</ul> 
<p>Each account must have an <a href="https://aws.amazon.com/vpc/" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Cloud</a> (Amazon VPC) with at least two private subnets in two different Availability Zones. For instruction, refer to <a href="https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html#create-vpc-and-other-resources" rel="noopener noreferrer" target="_blank">Create a VPC plus other VPC resources</a>. Make sure to create both VPCs in the same <a href="https://docs.aws.amazon.com/glossary/latest/reference/glos-chap.html#region" rel="noopener noreferrer" target="_blank">Region</a> you plan to apply this solution.</p> 
<p>A governance account is used for the sake of convenience, but it’s not strictly needed because Amazon SageMaker can be configured and managed in producer or consumer accounts.If you don’t have access to three accounts, you can still use this post to understand the key configurations required to implement a data mesh pattern with Amazon SageMaker Catalog while keeping your current data repositories and consumer applications unchanged.</p> 
<h3>Create a data repository in the producer account</h3> 
<p>First, create a sample dataset by following these instructions:</p> 
<ol> 
 <li>Open a text editor.</li> 
 <li>Paste the following text in a new file: 
  <div class="hide-language"> 
   <pre><code class="lang-code">name,stars
	oak,3
	maple,2
	birch,3
	willow,4
	pine,5
	mango,1
	neem,2
	banyan,5
	eucalyptus,3
	teak,2</code></pre> 
   <p></p>
  </div> </li> 
 <li>Save the file as <code>trees.csv</code>. This is your sample data file.</li> 
</ol> 
<p>After you create the sample dataset, create an S3 bucket and an AWS Glue database in the producer account, which will act as the data repository.</p> 
<p>Create the S3 bucket and upload the <code>trees.csv</code> file in the producer account:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">S3 console</a> in the producer account.</li> 
 <li>Create an S3 bucket. For instructions, refer to <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html" rel="noopener noreferrer" target="_blank">Creating a general purpose bucket</a>.</li> 
 <li>Upload to the S3 bucket the <code>trees.csv</code> sample data file that you created. For instructions, refer to <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/upload-objects.html" rel="noopener noreferrer" target="_blank">Uploading objects</a>.</li> 
</ol> 
<p>Create the AWS Glue database and table in the producer account:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/glue/home" rel="noopener noreferrer" target="_blank">Glue console</a> in the producer account.</li> 
 <li>In the navigation pane, under <strong>Data Catalog</strong>, choose <strong>Databases</strong>.</li> 
 <li>Choose <strong>Add database</strong>.</li> 
 <li>For <strong>Name</strong>, enter <code>collections</code>.</li> 
 <li>For <strong>Description</strong>, enter <code>This database contains collections of statistics for natural resources</code>.</li> 
 <li>Choose <strong>Create database</strong>.</li> 
 <li>In the navigation pane, under <strong>Data Catalog</strong>, choose <strong>Tables</strong>.</li> 
 <li>Choose <strong>Add table</strong>.</li> 
 <li>In the table creation guided procedure, enter the following input for <strong>Step 1: Set table properties</strong>: 
  <ol type="a"> 
   <li>For <strong>Name</strong>, enter <code>trees</code>.</li> 
   <li>For <strong>Database</strong>, select <code>collections</code>.</li> 
   <li>For <strong>Description</strong>, enter <code>This table captures ratings data related to the characteristics of various tree species</code>.</li> 
   <li>For <strong>Table format</strong>, select <strong>Standard AWS Glue table (default)</strong>.</li> 
   <li>For <strong>Select the type of source</strong>, select<strong> S3</strong>.</li> 
   <li>For <strong>Data location is specified in</strong>, select <strong>my account</strong>.</li> 
   <li>For <strong>Include path</strong>, enter <code>s3://&lt;bucket-name&gt;/&lt;prefix&gt;/</code> where <code>&lt;bucket-name&gt;</code> is the name of the S3 bucket you created earlier in this procedure and <code>&lt;prefix&gt;</code> is the optional prefix for the <code>trees.csv</code> file you uploaded.</li> 
   <li>For <strong>Data format</strong>, select <strong>CSV</strong>.</li> 
   <li>For <strong>Delimeter</strong>, select <strong>Comma (,)</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>For <strong>Step 2: Choose or define schema</strong>, enter the following: 
  <ol type="a"> 
   <li>For <strong>Schema</strong>, select <strong>Define or upload a schema</strong>.</li> 
   <li>Choose <strong>Edit schema as JSON</strong> and enter the following schema in the pop-up: 
    <div class="hide-language"> 
     <pre><code class="lang-code">[
  {
    "Name": "name",
    "Type": "string",
    "Parameters": {}
  },
  {
    "Name": "stars",
    "Type": "string",
    "Parameters": {}
  }
]</code></pre> 
    </div> </li> 
   <li>Choose <strong>Save</strong>.</li> 
   <li>Choose <strong>Next</strong>.</li> 
   <li>Choose <strong>Create</strong>.</li> 
  </ol> </li> 
</ol> 
<h3>Create a Lambda function in the consumer account</h3> 
<p>Create the Lambda function in the consumer account. This will simulate a data consumer application.First, in the consumer account create the IAM policy and the IAM role to be assigned to the Lambda function:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">IAM console</a> in the consumer account.</li> 
 <li>Create an IAM policy and name it <code>smus_consumer_athena_execution</code> by using the following policy. Make sure to replace placeholders <code>&lt;AWS_Region&gt;</code> and <code>&lt;AWS_account_ID_number&gt;</code> with your Region and consumer account ID number. You will replace the <code>&lt;workgroup_id&gt;</code> placeholder later. For IAM policy creation instructions, refer to <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html" rel="noopener noreferrer" target="_blank">Create IAM policies (console)</a>. 
  <div class="hide-language"> 
   <pre><code class="lang-code">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AthenaExecution",
            "Action": [
                "athena:StartQueryExecution",
                "athena:GetQueryExecution",
                "athena:GetQueryResults"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:athena:&lt;AWS_Region&gt;:&lt;AWS_account_ID_number&gt;:workgroup/&lt;workgroup_id&gt;"
        }
    ]
}</code></pre> 
  </div> </li> 
 <li>Create an IAM role for AWS Lambda service and name it <code>smus_consumer_lambda</code>. Assign to it the AWS managed permission <code>AWSLambdaBasicExecutionRole</code> and the permission named <code>smus_consumer_athena_execution</code> that you just created. For instructions, refer to <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html" rel="noopener noreferrer" target="_blank">Create a role to delegate permissions to an AWS service</a>.</li> 
</ol> 
<p>After the IAM role for the Lambda function is in place, you can create the Lambda function in the consumer account:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">Lambda console</a> in the consumer account.</li> 
 <li>In the navigation pane, choose <strong>Functions.</strong></li> 
 <li>Choose <strong>Create function</strong> and enter the following information<strong>:</strong> 
  <ol type="a"> 
   <li>For <strong>Function name</strong>, enter <code>consumer_function</code>.</li> 
   <li>For <strong>Runtime</strong>, select <strong>Python 3.14</strong>.</li> 
   <li>Expand <strong>Change default execution role</strong> section.</li> 
   <li>For <strong>Execution role</strong>, select <strong>Use an existing role</strong>.</li> 
   <li>For <strong>Existing role</strong>, select <code>smus_consumer_lambda</code>.</li> 
  </ol> </li> 
 <li>Choose <strong>Create function</strong>.</li> 
 <li>Under the <strong>Code</strong> tab, in the <strong>Code source</strong>, replace the existing code with the following: 
  <div class="hide-language"> 
   <pre><code class="lang-python">import boto3
import time
sts_client = boto3.client('sts')
role_arn = "&lt;role_arn&gt;"
session_name = "AthenaQuerySession"
catalog = "AwsDataCatalog"
database = "&lt;database_name&gt;"
workgroup = "&lt;workgroup_id&gt;"
query = "select * from "+catalog+"."+database+".trees"
def lambda_handler(event, context):
    # Assume SageMaker Unified Studio project role
    assumed_role_object = sts_client.assume_role(
        RoleArn=role_arn,
        RoleSessionName=session_name
    )
    # Get temporary credentials
    credentials = assumed_role_object['Credentials']
    # Create Athena client using temporary credentials
    athena = boto3.client(
        'athena',
        aws_access_key_id=credentials['AccessKeyId'],
        aws_secret_access_key=credentials['SecretAccessKey'],
        aws_session_token=credentials['SessionToken'],
        region_name='eu-west-1'
    )
    # Execute Athena Query
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={
            'Database': database,
            'Catalog': catalog
        },
        WorkGroup=workgroup
    )
    query_execution_id = response['QueryExecutionId']
    # Polling with exponential backoff
    wait_time = 0.25  # Start with 0.25 seconds
    max_wait = 8      # Maximum wait time of 8 seconds
    
    while True:
        result = athena.get_query_execution(QueryExecutionId=query_execution_id)
        state = result['QueryExecution']['Status']['State']
        if state in ['FAILED', 'CANCELLED']:
            raise Exception(f"Query {state}")
        elif state == 'SUCCEEDED':
            break
        elif state in ['QUEUED', 'RUNNING']:
            time.sleep(wait_time)
            wait_time = min(wait_time * 2, max_wait)  # Double wait time, cap at max_wait
    # Retrieve results
    results = athena.get_query_results(QueryExecutionId=query_execution_id)
    return results</code></pre> 
  </div> </li> 
 <li>Choose <strong>Deploy</strong>.</li> 
</ol> 
<p>The code provided for the Lambda function includes some placeholders that you will replace later, after you have the required information. Don’t test the Lambda function at this time because it will fail because of the presence of the placeholders.</p> 
<h3>Create a user with administrative access</h3> 
<p>Amazon SageMaker Unified Studio supports two distinct domain types: <a href="https://aws.amazon.com/iam/identity-center/" rel="noopener noreferrer" target="_blank">AWS IAM Identity Center</a> based domains and IAM based domains. At the time of writing this post, only IAM Identity Center based domains support multi-accounts association, therefore in this post you work with this type of domain that requires IAM Identity Center.</p> 
<p>In the governance account, you enable IAM Identity Center and create an administrative user to create and manage the Amazon SageMaker Unified Studio domain. Create a user with administrative access:</p> 
<ol> 
 <li>Enable IAM Identity Center in the governance account. For instructions, refer to <a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/get-set-up-for-idc.html" rel="noopener noreferrer" target="_blank">Enable IAM Identity Center</a>.</li> 
 <li>In IAM Identity Center in the governance account, grant administrative access to a user. For a tutorial about using the IAM Identity Center directory as your identity source, refer to <a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/quick-start-default-idc.html" rel="noopener noreferrer" target="_blank">Configure user access with the default IAM Identity Center directory</a>.</li> 
</ol> 
<p>Sign in as the user with administrative access:</p> 
<ul> 
 <li>To sign in with your IAM Identity Center user, use the sign-in URL that was sent to your email address when you created the IAM Identity Center user. For help signing in using an IAM Identity Center user, refer to <a href="https://docs.aws.amazon.com/signin/latest/userguide/iam-id-center-sign-in-tutorial.html" rel="noopener noreferrer" target="_blank">Sign in to your AWS access portal</a>.</li> 
</ul> 
<h3>Create a SageMaker Unified Studio domain</h3> 
<p>To create the Amazon SageMaker Unified Studio domain in the governance account refer to <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/create-domain-sagemaker-unified-studio-quick.html" rel="noopener noreferrer" target="_blank">Create a Amazon SageMaker Unified Studio domain – quick setup</a>.</p> 
<p>After your domain is created, you can navigate to the Amazon SageMaker Unified Studio portal (a browser-based web application) where you can use your data and configured tools for analytics and AI. Save the Amazon SageMaker Unified Studio portal URL because you will use this URL later.</p> 
<h2>Solution steps</h2> 
<p>Now that you have the prerequisites in place, you can complete the following ten high-level steps to implement the solution.</p> 
<h3>Associate the producer and consumer accounts to the Amazon SageMaker Unified Studio domain</h3> 
<p>Start by associating the producer and consumer accounts to the newly created Amazon SageMaker Unified Studio domain. When you associate your producer and consumer accounts to the domain, make sure to select <strong>IAM users and roles can access APIs and IAM users can log in to Amazon SageMaker Unified Studio</strong> in the <strong>AWS RAM share managed permission</strong> section. For step-by-step instructions, refer to <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/associated-accounts.html" rel="noopener noreferrer" target="_blank">Associated accounts in Amazon SageMaker Unified Studio</a>. If your AWS accounts are part of the same organization, your association requests are automatically accepted. However, if your AWS accounts aren’t part of the same organization, request association with the other AWS accounts in the governance account and then accept the association request in both the producer and consumer accounts.</p> 
<h3>Create two project profiles</h3> 
<p>Now, create two project profiles, one for the producer project and one for the consumer project.</p> 
<p>In Amazon SageMaker Unified Studio, a project profile defines an uber template for projects in your Amazon SageMaker domain. A project profile is a collection of blueprints that provides reusable <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> templates used to create project resources.</p> 
<p>A project profile is associated to a specific AWS account. This means, when a project is created the blueprints listed in the project profile are deployed in the associated AWS account. To use a project profile, you must enable its blueprints in the AWS account associated to the project profile.</p> 
<h4>Create the producer project profile</h4> 
<p>You’re going to create the producer project profile that is associated to the producer account. This project profile will be used to create the producer project. This profile includes by default the <code>Tooling</code> blueprint that creates resources for the project, including IAM user roles and security groups.</p> 
<p>Before creating the project profile, you will enable the <code>Tooling</code> blueprint in the producer account using the following procedure:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/datazone" rel="noopener noreferrer" target="_blank">SageMaker console</a> in the producer account.</li> 
 <li>In the navigation pane, choose <strong>Associated domains</strong>.</li> 
 <li>Select the domain you created while setting up.</li> 
 <li>On the <strong>Blueprints</strong> tab, choose <strong>Enable</strong> in the <strong>Tooling blueprint</strong> section as shown in the following image:</li> 
 <p> <img alt="SageMaker Unified Studios Tooling blueprint config: disabled status with Enable button for IAM roles &amp; AWS resource setup" class="alignnone wp-image-87942 size-full" height="233" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-2.png" style="margin: 10px 0px 10px 0px; border: 1px solid #cccccc;" width="1656" /></p> 
 <li>For <strong>Virtual private cloud (VPC)</strong> select your account VPC.</li> 
 <li>For <strong>Subnets</strong>, select at least two subnets in different Availability Zones.</li> 
 <li>Choose <strong>Enable blueprint</strong>.</li> 
</ol> 
<p>Proceed to creating the project profile in the governance account:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/datazone" rel="noopener noreferrer" target="_blank">SageMaker console</a> in the governance account.</li> 
 <li>In the navigation pane, choose <strong>Domains</strong>.</li> 
 <li>Select the domain you created as part of prerequisites.</li> 
 <li>Under the <strong>Project profiles</strong> tab, choose <strong>Create</strong> and enter the following information: 
  <ol type="a"> 
   <li>For <strong>Project profile name</strong>, enter <code>producer-project-profile</code>.</li> 
   <li>For <strong>Project profile creation options</strong>, select <strong>Custom create.</strong></li> 
   <li>DO NOT SELECT A BLUEPRINT for <strong>Blueprints </strong>because the <code>Tooling</code> blueprint is included by default in any project profile.</li> 
   <li>For <strong>Account</strong>, select <strong>Provide an account ID</strong>.</li> 
   <li>For <strong>Account ID</strong>, enter the producer account ID.</li> 
   <li>For <strong>Region</strong>, select <strong>Provide region name</strong> and then select the Region in which you’re working.</li> 
   <li>For <strong>Authorization</strong>, select <strong>Allow all users and groups</strong>.</li> 
   <li>For <strong>Project profile readiness</strong>, select <strong>Enable project profile on creation</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Create project profile</strong>.</li> 
</ol> 
<h4>Create a consumer project profile</h4> 
<p>You also create a consumer project profile and associate it to the consumer account. This profile will be used to create the consumer project. The consumer project profile includes the <code>LakeHouseDatabase</code> blueprint, which is needed to create a lakehouse environment with an AWS Glue database for data management and an Amazon Athena workgroup for querying. The <code>Tooling</code> blueprint is included by default in the project profile.</p> 
<p>Before creating the project profile, enable the <code>Tooling</code> and <code>LakeHouseDatabase</code> blueprints in the consumer account:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/datazone" rel="noopener noreferrer" target="_blank">SageMaker console</a> in the consumer account.</li> 
 <li>In the navigation pane, choose <strong>Associated domains</strong>.</li> 
 <li>Select the domain you created as part of prerequisites.</li> 
 <li>On the <strong>Blueprints</strong> tab, choose <strong>Enable</strong> in the <strong>Tooling blueprint</strong> section.</li> 
 <li>For <strong>Virtual private cloud (VPC)</strong> select your account VPC.</li> 
 <li>For <strong>Subnets</strong>, select at least two subnets in different Availability Zones.</li> 
 <li>Choose <strong>Enable blueprint</strong>.</li> 
 <li>In the navigation pane, choose <strong>Associated domains</strong>.</li> 
 <li>Select the domain you created as part of prerequisites.</li> 
 <li>Under the <strong>Blueprints</strong> tab, select the <code>LakeHouseDatabase</code> blueprint.</li> 
 <li>Choose <strong>Enable</strong>.</li> 
 <li>Choose <strong>Enable blueprint</strong>.</li> 
</ol> 
<p>After blueprints are enabled in the consumer account, you can proceed creating the project profile:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/datazone" rel="noopener noreferrer" target="_blank">SageMaker console</a> in the governance account.</li> 
 <li>In the navigation pane, choose <strong>Domains</strong>.</li> 
 <li>Select the domain you created as part of prerequisites.</li> 
 <li>Under <strong>Project profiles</strong> tab choose <strong>Create</strong> and enter the following information: 
  <ol type="a"> 
   <li>For <strong>Project profile name</strong>, enter <code>consumer-project-profile</code>.</li> 
   <li>For <strong>Project profile creation options</strong>, select <strong>Custom create.</strong></li> 
   <li>For <strong>Blueprints</strong>, select <code>LakeHouseDatabase</code>.</li> 
   <li>For <strong>Account</strong>, select <strong>Provide an account ID</strong>.</li> 
   <li>For <strong>Account ID</strong>, enter the consumer account ID.</li> 
   <li>For <strong>Region</strong>, select <strong>Provide region name</strong> and then select the Region you are working.</li> 
   <li>For <strong>Authorization</strong>, select <strong>Allow all users and groups</strong>.</li> 
   <li>For <strong>Project profile readiness</strong>, select <strong>Enable project profile on creation</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Create project profile</strong>.</li> 
</ol> 
<h3>Create SageMaker Unified Studio producer and consumer projects</h3> 
<p>In Amazon SageMaker Unified Studio, a project is a boundary within a domain where you can collaborate with other users to work on a business use case. In projects, you can create and share data and resources.To create producer and consumer projects in Amazon SageMaker Unified Studio use the following instructions:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> dropdown list.</li> 
 <li>Choose <strong>Create project</strong> and enter the following information: 
  <ol type="a"> 
   <li>For <strong>Project name</strong>, enter <code>Producer</code>.</li> 
   <li>For <strong>Project profile</strong>, select <code>producer-project-profile</code>.</li> 
  </ol> </li> 
 <li>Choose <strong>Continue.</strong></li> 
 <li>Choose <strong>Continue.</strong></li> 
 <li>Choose <strong>Create project.</strong></li> 
</ol> 
<p>After you’ve created the <code>Producer</code> project, note in a text file the <strong>Project role ARN</strong> that is displayed in the <strong>Project overview</strong>. The following image is shown for reference. The project role name is the string that follows <code>arn:aws:iam::&lt;account_ID&gt;:role/</code> in the project role <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener noreferrer" target="_blank">Amazon Resource Name</a> (ARN). You will use both project role name and ARN later.</p> 
<p><img alt="SageMaker Producer project overview: active status, files listed, S3 location &amp; IAM role ARN displayed in project details tab" class="alignnone wp-image-87943 size-full" height="759" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-3.png" width="1912" /></p> 
<p>Repeat the preceding procedure to create the <code>Consumer</code> project. Be sure to enter <code>Consumer</code> for <strong>Project name</strong> and then select <code>consumer-project-profile</code> for <strong>Project profile</strong>. After it’s created, note the <strong>Project role ARN</strong> in a text file. The project role name is the string that follows <code>arn:aws:iam::&lt;account_ID&gt;:role/</code> in the project role ARN. You will use both project role name and ARN later.</p> 
<h3>Bring your own data from the producer account</h3> 
<p>Bring your own data to the Amazon SageMaker Unified Studio <code>Producer</code> project. AWS provides several options to achieve this onboarding. The first option is automated onboarding in Amazon SageMaker lakehouse, in which you ingest the Amazon SageMaker lakehouse metadata of datasets into Amazon SageMaker Catalog. With this option, you can onboard your Amazon SageMaker lakehouse data as part of creating a new Amazon SageMaker Unified Studio domain or for an existing domain.</p> 
<p>For more information about automated onboarding of Amazon SageMaker lakehouse data, refer to <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/data-onboarding.html" rel="noopener noreferrer" target="_blank">Onboarding data in Amazon SageMaker Unified Studio</a>. As other options, you can bring in existing resources to your Amazon SageMaker Unified Studio project by using the <strong>Data</strong> and <strong>Compute</strong> pages in your project, or by using scripts provided in GitHub. For more information about using the <strong>Data</strong> and <strong>Compute</strong> pages or about using scripts, refer to <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/bring-resources-scripts.html" rel="noopener noreferrer" target="_blank">Bringing existing resources into Amazon SageMaker Unified Studio</a>. In this post, you will use Amazon SageMaker lakehouse capabilities to import your <code>trees</code> AWS Glue table into the <code>Producer</code> project.</p> 
<h4>Register the Amazon S3 location for the table</h4> 
<p>To use Lake Formation permissions for fine-grained access control to the <code>trees</code> table, you need to register in Lake Formation the Amazon S3 location of the <code>trees</code> table. To do that, complete the following actions:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lakeformation" rel="noopener noreferrer" target="_blank">Lake Formation console</a> in the producer account.</li> 
 <li>In the navigation pane under <strong>Administration</strong>, choose <strong>Data lake locations</strong>.</li> 
 <li>Choose <strong>Register location</strong> and enter the following information: 
  <ol type="a"> 
   <li>For <strong>S3 URI</strong>, enter <code>s3://&lt;bucket-name&gt;/&lt;prefix&gt;/</code> where <code>&lt;bucket-name&gt;</code> is the name of the S3 bucket you created in the prerequisites and <code>&lt;prefix&gt;</code> is the optional prefix for the <code>trees.csv</code> file you uploaded as part of the prerequisite.</li> 
   <li>For <strong>IAM role</strong>, select <code>AWSServiceRoleForLakeFormationDataAccess</code>.</li> 
   <li>For <strong>Permission mode</strong>, select <strong>Lake Formation</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Register location</strong>.</li> 
</ol> 
<h4>Grant <code>Producer</code> project role permissions on the database</h4> 
<p>Grant database access to the IAM role that is associated with your <code>Producer</code> project. This role is called the project role, and it was created in IAM upon project creation.</p> 
<p>To access the AWS Glue Data Catalog <code>collections</code> database from the <code>Producer</code> project in the Amazon SageMaker Unified Studio, complete the following actions:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lakeformation" rel="noopener noreferrer" target="_blank">Lake Formation console</a> in the producer account.</li> 
 <li>In the navigation pane under <strong>Data Catalog</strong>, choose <strong>Databases</strong>.</li> 
 <li>Choose the <code>collections</code> database.</li> 
 <li>From the <strong>Actions</strong> menu, choose <strong>Grant</strong> and enter the following information: 
  <ol type="a"> 
   <li>For <strong>IAM users and roles</strong>, select your <code>Producer</code> project’s role name. This is the string starting with <code>datazone_usr_role_</code> that is part of the <code>Producer</code> project role ARN that you noted in step 3 “Create SageMaker Unified Studio producer and consumer projects”.</li> 
   <li>For <strong>Database permissions</strong>, select <strong>Describe</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Grant</strong>.</li> 
</ol> 
<h4>Grant <code>Producer</code> project role permissions on the table</h4> 
<p>Grant <code>trees</code> table access to the IAM role that is associated with your <code>Producer</code> project. To grant these permissions use the following instructions:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lakeformation" rel="noopener noreferrer" target="_blank">Lake Formation console</a> in the producer account.</li> 
 <li>In the navigation pane under <strong>Data Catalog</strong>, choose <strong>Tables and MVs</strong>.</li> 
 <li>Select the <code>trees</code> table.</li> 
 <li>From the <strong>Actions</strong> menu, choose <strong>Grant</strong> and enter the following information: 
  <ol type="a"> 
   <li>For <strong>IAM users and roles</strong>, select your <code>Producer</code> project’s role. This is the string starting with <code>datazone_usr_role_</code> that is part of the <code>Producer</code>project role ARN that you noted in step 3 “Create SageMaker Unified Studio producer and consumer projects”.</li> 
   <li>For <strong>Table permissions</strong>, select <strong>Select</strong> and <strong>Describe</strong>.</li> 
   <li>For <strong>Grantable permissions</strong>, select <strong>Select</strong> and <strong>Describe</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Grant</strong>.</li> 
</ol> 
<h4>Revoke any existing permissions of <code>IAMAllowedPrincipals</code></h4> 
<p>You must revoke the <code>IAMAllowedPrincipals</code> group permissions on both the database and table to enforce Lake Formation permission for access. For more information, refer to <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/revoking-permssions-console-all.html" rel="noopener noreferrer" target="_blank">Revoking permission using the Lake Formation console</a>.</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lakeformation" rel="noopener noreferrer" target="_blank">Lake Formation console</a> in the producer account.</li> 
 <li>In the navigation pane under <strong>Permission</strong>, choose <strong>Data permissions.</strong></li> 
 <li>Select the entries where <strong>Principal</strong> is set to <code>IAMAllowedPrincipals</code> and <strong>Resource</strong> is set to <code>collections</code> or <code>trees</code> as in the following image:</li> 
 <p> <img alt="Data permissions table: 2 of 5 IAMAllowedPrincipals entries selected. All permissions granted for collections DB &amp; trees table" class="alignnone wp-image-87944 size-full" height="349" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-4.png" width="1544" /></p> 
 <li>Choose <strong>Revoke</strong>.</li> 
 <li>Enter <code>revoke</code>.</li> 
 <li>Choose <strong>Revoke</strong> again.</li> 
</ol> 
<h3>Verify that data is available in the <code>Producer</code> project</h3> 
<p>Verify that your <code>collections</code> database and <code>trees</code> table are accessible in the <code>Producer</code> project:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> drop-down menu and choose the <code>Producer</code> project.</li> 
 <li>In the navigation pane under <strong>Overview</strong>, choose <strong>Data</strong>.</li> 
 <li>Choose <strong>Lakehouse</strong>.</li> 
 <li>Choose <strong>AwsDataCatalog</strong>.</li> 
 <li>Choose <code>collections</code>.</li> 
 <li>Choose <strong>tables</strong>.</li> 
 <li>Choose the three-dot action menu next to your <code>trees</code> table and choose <strong>Preview data</strong>, as shown in the following image.<br /> <img alt="AWS Data Catalog interface: collections database in Lakehouse with trees table, presenting preview/notebook/drop options" class="alignnone wp-image-87945 size-full" height="649" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-5.png" width="1206" /></li> 
 <li>You’ll find data from the <code>trees</code> table as shown in the following image.<br /> <img alt="Query Editor showing SQL query on trees table with results: oak (3 stars), maple (2), birch (3). Red arrow highlights output" class="alignnone wp-image-87946 size-full" height="734" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-6.png" width="1371" /></li> 
</ol> 
<h3>Create Amazon SageMaker Catalog asset</h3> 
<p>Even if it’s accessible in the project, to work with the <code>trees</code> table in Amazon SageMaker Catalog, you need to register the data source and create an Amazon SageMaker Catalog asset:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> dropdown list and choose the <code>Producer</code> project.</li> 
 <li>On the project page, under <strong>Project catalog </strong>in the navigation pane, choose <strong>Data sources</strong>.</li> 
 <li>Choose <strong>Create Data Source</strong> and make the following selections: 
  <ol type="a"> 
   <li>For <strong>Name</strong>, enter <code>collections</code>.</li> 
   <li>For <strong>Data source type</strong>, select <strong>AWS Glue (Lakehouse)</strong>.</li> 
   <li>For <strong>Database name</strong>, select <code>collections</code>.</li> 
   <li>Choose <strong>Next</strong>.</li> 
   <li>Choose <strong>Next</strong>.</li> 
   <li>Choose <strong>Next</strong>.</li> 
   <li>Choose <strong>Create</strong>.</li> 
  </ol> </li> 
 <li>After the data source is created, you will be in the <code>collections</code> data source page, choose <strong>Run</strong>. This will import metadata and create the Amazon SageMaker Catalog asset.</li> 
 <li>In the <code>collections</code> data source, on the <strong>Data source runs</strong> tab, you’ll find your run marked as <strong>Completed</strong> and the <code>trees</code> asset <strong>Successfully created</strong>, as shown in the following image:<br /> <img alt="Producer project Assets page: Inventory tab presenting trees Glue Table asset with red arrows highlighting navigation &amp; selection" class="alignnone wp-image-87947 size-full" height="303" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-7.png" width="940" /></li> 
</ol> 
<h3>Publish the data asset in the Amazon SageMaker Catalog</h3> 
<p>Publishing a data asset manually is a one-time operation that you need to perform to allow others to access the data asset through the catalog:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> dropdown list and choose the <code>Producer</code> project.</li> 
 <li>On the project page under <strong>Project catalog</strong>, choose <strong>Assets</strong>.</li> 
 <li>Select your <code>trees</code> data asset that is available on the <strong>Inventory</strong> tab. The following image is shown for reference.<br /> <img alt="Assets Inventory page: trees Glue Table listed in Producer project with navigation arrows highlighting menu selection" class="alignnone wp-image-87948 size-full" height="645" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-8.png" width="1793" /></li> 
 <li>(Optional) If automated metadata generation is enabled when the data source is created, metadata for assets (such as the asset business name) is available to review and accept or reject. You can either choose <strong>Accept All</strong> or <strong>Reject All</strong> in the <strong>Automated Metadata Generation</strong> banner.</li> 
 <li>Choose <strong>Publish Asset</strong>. The following image is shown for reference.<br /> <img alt="Asset overview: Agricultural Crop Yield dataset with automated metadata banner, ACCEPT ALL &amp; PUBLISH ASSET buttons highlighted" class="alignnone wp-image-87949 size-full" height="715" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-9.png" width="1792" /></li> 
 <li>Choose <strong>Publish Asset</strong>.</li> 
</ol> 
<h3>Subscribe to the data asset in the Amazon SageMaker Catalog</h3> 
<p>To consume data assets in the <code>Consumer</code> project, subscribe to the data asset by creating a subscription request:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> dropdown list and choose <code>Consumer</code> project.</li> 
 <li>On the <strong>Discover </strong>menu, choose <strong>Catalog</strong>.</li> 
 <li>Enter <code>trees</code> in the search box and then select the data asset returned from the search. If in step 7 “Publish the data asset in the Amazon SageMaker Catalog” you chose <strong>Accept All</strong> in the <strong>Automated Metadata Generation </strong>banner, your data asset will have a different business name generated by the <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/autodoc.html" rel="noopener noreferrer" target="_blank">automated metadata recommendations</a> feature. The data asset technical name is <code>trees</code>. For reference, refer to the following image.<br /> <img alt="Data Catalog search: 'trees' query shows Agricultural Crop Yield dataset with browse assets &amp; data products options" class="alignnone wp-image-87950 size-full" height="367" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-10.png" width="1381" /></li> 
 <li>Choose <strong>Subscribe</strong>.</li> 
 <li>For <strong>Comment</strong>, enter a justification such as <code>This data asset is needed for model training purposes</code>.</li> 
 <li>Choose <strong>Subscribe</strong> again.</li> 
</ol> 
<p>By default, asset subscription requests require manual approval by a data owner. However, if the requester in the <code>Consumer</code> project is also a member of the <code>Producer</code> project, the subscription request is automatically approved. For information about approving subscription requests, refer to <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/approve-reject-subscription-request.html" rel="noopener noreferrer" target="_blank">Approve or reject a subscription request in Amazon SageMaker Unified Studio</a>.</p> 
<h3>Configure your Lambda IAM role to access the subscribed data access</h3> 
<p>To enable your Lambda function access to the subscribed data asset, you need to allow the Lambda function to assume the <code>Consumer</code> project role. To do this, edit the <code>Consumer</code> project’s IAM role trust relationship:</p> 
<ol> 
 <li>Navigate to the <a href="https://console.aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">IAM console</a> in the consumer account.</li> 
 <li>In the navigation pane under <strong>Access management</strong>, choose <strong>Roles</strong>.</li> 
 <li>Select the <code>Consumer</code> project’s IAM role. This is the string starting with <code>datazone_usr_role_</code> that is part of the <code>Consumer</code> project role ARN that you noted in step 3 “Create SageMaker Unified Studio producer and consumer projects”.</li> 
 <li>Under the <strong>Trust relationships</strong> tab, choose <strong>Edit trust policy</strong>.</li> 
 <li>For backup reasons, make a copy of the existing trust policy in a text file.</li> 
 <li>In the <strong>Edit trust policy</strong> window, add the following statement to the existing trust policy without removing or overwriting other existing statements in the trust policy. Be sure to replace the placeholder <code>&lt;account_id&gt;</code> with your consumer AWS account ID. 
  <div class="hide-language"> 
   <pre><code class="lang-json">{
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::&lt;account_id&gt;:role/smus_consumer_lambda"
    },
    "Action": [
        "sts:AssumeRole"
    ]
}	</code></pre> 
  </div> <p> <img alt="IAM trust policy editor: JSON code with red arrow highlighting AWS principal ARN for smus_consumer_lambda role" class="alignnone wp-image-87951 size-full" height="666" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-11.png" style="margin: 10px 0px 10px 0px; border: 1px solid #cccccc;" width="1169" /></p></li> 
 <li>Choose <strong>Update policy</strong>.</li> 
</ol> 
<h3>Test the Lambda function’s access to the subscribed data asset</h3> 
<p>Before you can test your Lambda function, you need to replace placeholders in the function code and in the IAM policy. There are three placeholders to be replaced: <code>&lt;role_arn&gt;</code>, <code>&lt;database_name&gt;</code> and <code>&lt;workgroup_id&gt;</code>. For <code>&lt;role_arn&gt;</code>, you already have the actual value, which is the <code>Consumer</code> project’s role ARN that you noted in step 3 “Create SageMaker Unified Studio producer and consumer projects”. The next sections provide instructions to retrieve values for the other placeholders.</p> 
<h4>Retrieve the AWS Glue Data Catalog database name</h4> 
<p>You need to find the name of the AWS Glue Data Catalog database that was created along with the <code>Consumer</code> project. You will then use this value to replace the <code>&lt;database_name&gt;</code> placeholder in the <code>consumer_function</code> Lambda function code. To retrieve the AWS Glue Data Catalog database name, follow these instructions:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> dropdown list and choose <code>Consumer</code> project.</li> 
 <li>On the project page, under <strong>Overview</strong>, choose <strong>Data</strong>.</li> 
 <li>Choose <strong>Lakehouse</strong>.</li> 
 <li>Choose <strong>AwsDataCatalog</strong>.</li> 
 <li>Copy the name of the database. It should be an alphanumerical string starting with <code>glue_db</code>, as in the following image:</li> 
 <p> <img alt="Consumer project Data page: Lakehouse &gt; AwsDataCatalog &gt; glue_db database navigation with tables &amp; views expandable sections" class="alignnone wp-image-87952 size-full" height="294" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-12.png" width="1084" /> </p>
</ol> 
<h4>Retrieve the Athena workgroup ID</h4> 
<p>You need to find the ID of the Athena workgroup that was created along with the <code>Consumer</code> project. You will then use this value to replace the <code>&lt;workgroup_id&gt;</code> placeholder in the <code>consumer_function</code> Lambda function code and in the <code>smus_consumer_athena_execution</code> IAM policy. Use the following instructions to retrieve the Athena workgroup ID:</p> 
<ol> 
 <li>Access the Amazon SageMaker Unified Studio portal.</li> 
 <li>Choose the <strong>Select a project</strong> dropdown list and choose <code>Consumer</code> project.</li> 
 <li>On the project page, under <strong>Overview</strong>, choose <strong>Compute</strong>.</li> 
 <li>Under the <strong>SQL analytics</strong> tab, select <strong>project.athena</strong>, as in the following image:<br /> <img alt="Consumer project Compute page: SQL analytics tab showing project.athena resource with Available status and navigation arrows" class="alignnone wp-image-87953 size-full" height="508" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-13.png" width="1074" /></li> 
 <li>Copy the <strong>Workgroup ARN</strong> and save to a text file. The Athena workgroup ID is the string that follows <code>arn:aws:athena:&lt;region&gt;:&lt;account_ID&gt;:workgroup/</code> in the Workgroup ARN.</li> 
</ol> 
<h4>Replace placeholder in the <code>smus_consumer_athena_execution</code> IAM policy</h4> 
<p>To replace the <code>&lt;workgroup_id&gt;</code> placeholder in the <code>smus_consumer_athena_execution</code> IAM policy, use the following procedure:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">IAM console</a> in the consumer account.</li> 
 <li>In the navigation pane, choose <strong>Policies</strong>.</li> 
 <li>In the search field enter <code>smus_consumer_athena_execution</code>.</li> 
 <li>Select the <code>smus_consumer_athena_execution</code> policy.</li> 
 <li>Choose <strong>Edit</strong>.</li> 
 <li>Replace <code>&lt;workgroup_id&gt;</code> with the value you noted earlier.</li> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>Choose <strong>Save changes</strong>.</li> 
</ol> 
<h4>Replace placeholders in the Lambda function code and test it</h4> 
<p>In this section, you will replace the <code>&lt;role_arn&gt;</code>, <code>&lt;database_name&gt;</code> and <code>&lt;workgroup_id&gt;</code> placeholders in the <code>consumer_function</code> Lambda function code, and then you can test the function ability to access data of the <code>trees</code> table.</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">Lambda console</a> in the consumer account.</li> 
 <li>In the navigation pane, choose <strong>Functions</strong>.</li> 
 <li>Select <code>consumer_function</code>.</li> 
 <li>Under the <strong>Code</strong> tab, replace <code>&lt;role_arn&gt;</code>, <code>&lt;database_name&gt;</code> and <code>&lt;workgroup_id&gt;</code> placeholders with the respective values you noted earlier.</li> 
 <li>Choose <strong>Deploy</strong>.</li> 
 <li>Under the <strong>Test</strong> tab, for <strong>Event name</strong>, enter <code>mytest</code>.</li> 
 <li>Choose <strong>Test</strong>.</li> 
 <li>Choose <strong>Details</strong> in the green banner titled <strong>Executing function</strong> that appears after the execution is completed.</li> 
 <li>The execution log reports the <code>trees</code> table content, as shown in the following image:<br /> <img alt="Lambda test results: consumer_function succeeded with JSON output showing VarCharValue 'ok' and '3', execution details available" class="alignnone wp-image-87954 size-full" height="465" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/BDB-5457-image-14.png" style="margin: 10px 0px 10px 0px; border: 1px solid #cccccc;" width="1379" /></li> 
</ol> 
<p>If your Lambda function execution fails due to timeout, change the function timeout setting as follows:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">Lambda console</a> in the consumer account.</li> 
 <li>In the navigation pane, choose <strong>Functions</strong>.</li> 
 <li>Select <code>consumer_function</code>.</li> 
 <li>Under the <strong>Configuration</strong> tab, choose <strong>Edit</strong>.</li> 
 <li>For <strong>Timeout</strong>, enter 15 sec or a greater value.</li> 
 <li>Choose <strong>Save</strong>.</li> 
</ol> 
<p>After increasing the timeout, test the function again.</p> 
<h2>Clean up</h2> 
<p>If you no longer need the resources you created as you followed this post, delete them to prevent incurring additional charges. Start by deleting your Amazon SageMaker Unified Studio domain in the governance account. For more information, refer to <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/delete-domain.html" rel="noopener noreferrer" target="_blank">Delete domains</a>.</p> 
<p>To remove the AWS Glue <code>collections</code> database from the producer account, follow these steps:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/glue/home" rel="noopener noreferrer" target="_blank">Glue console</a> in the producer account.</li> 
 <li>In the navigation pane under <strong>Data Catalog</strong>, choose <strong>Databases</strong>.</li> 
 <li>Select the <code>collections</code> database.</li> 
 <li>Choose <strong>Delete</strong>.</li> 
 <li>Choose <strong>Delete</strong>.</li> 
</ol> 
<p>To remove the S3 bucket from the producer account, empty the bucket and then you can delete the bucket. For information about emptying the bucket, refer to <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/empty-bucket.html" rel="noopener noreferrer" target="_blank">Emptying a general purpose bucket</a>. For information about deleting the bucket, refer to <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-bucket.html" rel="noopener noreferrer" target="_blank">Deleting a general purpose bucket</a>.</p> 
<p>To remove the Lambda function from the consumer account, follow these steps:</p> 
<ol> 
 <li>Access the <a href="https://console.aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">Lambda console</a> in the consumer account.</li> 
 <li>In the navigation pane, choose <strong>Functions</strong>.</li> 
 <li>Select the <code>consumer_function</code> Lambda function.</li> 
 <li>Choose the <strong>Actions</strong> menu and then choose <strong>Delete function</strong>.</li> 
 <li>Enter <code>confirm</code>.</li> 
 <li>Choose <strong>Delete</strong>.</li> 
</ol> 
<p>To complete the cleanup, delete the IAM role named <code>smus_consumer_lambda</code>, then delete the IAM policy named <code>smus_consumer_athena_execution</code> in the consumer account. For information about removing a IAM role, refer to <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_manage_delete.html" rel="noopener noreferrer" target="_blank">Delete roles or instance profiles</a>. For information about removing an IAM policy, refer to <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-delete.html" rel="noopener noreferrer" target="_blank">Delete IAM policies</a>.</p> 
<h2>Conclusion</h2> 
<p>In this post, we covered adopting Amazon SageMaker Catalog for data governance without rearchitecting your existing applications and data repositories. We walked through how to onboard existing data in Amazon SageMaker Unified Studio, then publish it in a catalog, and then subscribe and consume the data from resources deployed outside the context of an Amazon SageMaker Unified Studio project. This solution can help you accelerate your implementation of a data mesh pattern with Amazon SageMaker Catalog to publish, find, and access data securely in your organization.</p> 
<p>For more information, refer to <a href="https://docs.aws.amazon.com/next-generation-sagemaker/latest/userguide/what-is-sagemaker.html#guide-to-sagemaker" rel="noopener noreferrer" target="_blank">What is Amazon SageMaker?</a> and work through the <a href="https://catalog.us-east-1.prod.workshops.aws/workshops/06dbe60c-3a94-463e-8ac2-18c7f85788d4/en-US" rel="noopener noreferrer" target="_blank">Amazon SageMaker Workshop</a> to try the unified experience for data, analytics, and AI.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-thumbnail wp-image-87958" height="107" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/Paolo-Romagno_revli-100x107.png" width="100" />
  </div> 
  <h3 class="lb-h4">Paolo Romagnoli</h3> 
  <p><a href="https://www.linkedin.com/in/paoloromagnoli/" rel="noopener noreferrer" target="_blank">Paolo</a> is a Senior Solutions Architect at AWS for Energy and Utilities. With 20+ years of experience in designing and building enterprise solutions, he works with global energy customers to design solutions to address customers’ business and technical needs. He is passionate about technology and enjoys running.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-thumbnail wp-image-87964" height="134" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/13/Photo_Joel-100x134.jpg" width="100" />
  </div> 
  <h3 class="lb-h4">Joel Farvault</h3> 
  <p><a href="https://www.linkedin.com/in/joel-farvault-4332331/" rel="noopener noreferrer" target="_blank">Joel</a> is a Principal Specialist SA Analytics for AWS with 25 years’ experience working on enterprise architecture, data governance and analytics. He uses his experience to advise customers on their data strategy and technology foundations.</p> 
 </div> 
</footer>

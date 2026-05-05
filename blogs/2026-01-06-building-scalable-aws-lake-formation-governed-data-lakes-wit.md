---
title: "Building scalable AWS Lake Formation governed data lakes with dbt and Amazon Managed Workflows for Apache Airflow"
url: "https://aws.amazon.com/blogs/big-data/building-scalable-aws-lake-formation-governed-data-lakes-with-dbt-and-amazon-managed-workflows-for-apache-airflow/"
date: "Tue, 06 Jan 2026 22:37:30 +0000"
author: "Abhilasha Agarwal"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p>Organizations often struggle with building scalable and maintainable data lakes—especially when handling complex data transformations, enforcing data quality, and monitoring compliance with established governance. Traditional approaches typically involve custom scripts and disparate tools, which can increase operational overhead and complicate access control. A scalable, integrated approach is needed to simplify these processes, improve data reliability, and support enterprise-grade governance.</p> 
<p><a href="https://airflow.apache.org/" rel="noopener noreferrer" target="_blank">Apache Airflow</a> has emerged as a powerful solution for orchestrating complex data pipelines in the cloud. <a href="https://aws.amazon.com/managed-workflows-for-apache-airflow/" rel="noopener noreferrer" target="_blank">Amazon Managed Workflows for Apache Airflow (MWAA</a>) extends this capability by providing a fully managed service that eliminates infrastructure management overhead. This service enables teams to focus on building and scaling their data workflows while AWS handles the underlying infrastructure, security, and maintenance requirements.</p> 
<p><a href="https://docs.getdbt.com/" rel="noopener noreferrer" target="_blank">dbt</a> enhances data transformation workflows by bringing software engineering best practices to analytics. It enables analytics engineers to transform warehouse data using familiar SQL select statements while providing essential features like version control, testing, and documentation. As part of the ELT (Extract, Load, Transform) process, dbt handles the transformation phase, working directly within a data warehouse to enable efficient and reliable data processing. This approach allows teams to maintain a single source of truth for metrics and business definitions while enabling data quality through built-in testing capabilities.</p> 
<p>In this post, we show how to build a governed data lake that uses modern data tools and AWS services.</p> 
<h2>Solution overview</h2> 
<p>We explore a comprehensive solution that includes:</p> 
<ul> 
 <li>A metadata-driven framework in MWAA that dynamically generates directed acyclic graphs (DAGs), significantly improving pipeline scalability and reducing maintenance overhead.</li> 
 <li>dbt with <a href="https://docs.aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a> adapter to implement modular, SQL-based data transformations directly on a data lake, enabling well-structured, and thoroughly tested transformations.</li> 
 <li>An automated framework that proactively identifies and segregates problematic records, maintaining the integrity of data assets.</li> 
 <li><a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a> to implement fine-grained access controls for Athena tables, ensuring proper data governance and security throughout a data lake environment.</li> 
</ul> 
<p>Together, these components create a robust, maintainable, and secure data management solution suitable for enterprise-scale deployments.</p> 
<p>The following architecture illustrates the components of the solution.</p> 
<p><img class="alignnone wp-image-86110 size-full" height="618" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/17/BDB-4834-arch-diag.png" width="1089" /></p> 
<p>The workflow contains the following steps:</p> 
<ol> 
 <li>Multiple data sources (PostgreSQL, MySQL, SFTP) push data to an <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon S3</a> raw bucket</li> 
 <li>S3 event triggers <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> Function</li> 
 <li>Lambda function triggers the MWAA DAG to convert file formats to parquet</li> 
 <li>Data is stored in Amazon S3 formatted bucket under formatted_stg prefix</li> 
 <li>Crawler crawls the data in formatted_stg prefix in the formatted bucket and creates catalog tables</li> 
 <li>dbt using Athena adapter processes the data and puts the processed data after data quality checks under formatted prefix in Formatted bucket</li> 
 <li>dbt using Athena adapter can perform further transformations on the formatted data and put the transformed data in Published bucket</li> 
</ol> 
<h2>Prerequisites</h2> 
<p>To implement this solution, the following prerequisites need to be met.</p> 
<ul> 
 <li>An AWS account with create/operate access for the following AWS services: 
  <ul> 
   <li><a href="https://docs.aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon S3</a></li> 
   <li><a href="https://docs.aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a></li> 
   <li><a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a></li> 
   <li><a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a></li> 
   <li><a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a></li> 
   <li><a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a></li> 
   <li><a href="https://docs.aws.amazon.com/mwaa/latest/userguide/what-is-mwaa.html?refid=a074e8bd-fe9a-4ee3-ad49-f731a39ed149" rel="noopener noreferrer" target="_blank">Amazon Managed Workflows for Apache Airflow (MWAA)</a></li> 
  </ul> </li> 
</ul> 
<h2>Deploy the solution</h2> 
<p>For this solution, we provide an <a href="http://aws.amazon.com/cloudformation" rel="noopener noreferrer" target="_blank">AWS CloudFormation (CFN)</a> template that sets up the services included in the architecture, to enable repeatable deployments.</p> 
<p><strong>Note:</strong></p> 
<ul> 
 <li>US-EAST-1 Region is required for the deployment.</li> 
 <li>Deploying this solution will involve costs associated with AWS services.</li> 
</ul> 
<p>To deploy the solution, complete the following steps:</p> 
<ol> 
 <li>Before deploying the stack, open the <a href="https://us-east-1.console.aws.amazon.com/lakeformation/home?region=us-east-1#administrative-roles-and-tasks" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a> console. Add your console role as a <strong>Data Lake Administrator</strong> and choose <strong>Confirm</strong> to save the changes.<img class="alignnone size-full wp-image-86109" height="605" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-2.png" width="1336" /></li> 
 <li><a href="https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/BDB-4834/dbt-mwaa-blog-cfn.yaml" rel="noopener noreferrer" target="_blank">Download</a> the CloudFormation template.<br /> After the file is downloaded to the local machine, follow the steps below to deploy the stack using this template:<p></p> 
  <ol type="a"> 
   <li>Open the <a href="https://console.aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation Console</a>.</li> 
   <li>Choose <strong>Create stack</strong> and choose <strong>With new resources (standard)</strong>.</li> 
   <li>Under <strong>Specify template</strong>, select <strong>Upload a template file</strong>.</li> 
   <li>Select <strong>Choose file</strong> and upload the CFN template that was downloaded earlier.</li> 
   <li>Choose <strong>Next</strong> to proceed.</li> 
  </ol> <p><img class="alignnone size-full wp-image-86108" height="723" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-3.png" width="1669" /></p></li> 
 <li>Enter a stack name (for example, bdb4834-data-lake-blog-stack) and configure the parameters (bdb4834-MWAAClusterName can be left as the default value and update SNSEmailEndpoints with your email address), then choose <strong>Next</strong>.<br /> <img class="alignnone size-full wp-image-86107" height="614" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-4.png" width="1214" /></li> 
 <li>Select “<strong>I acknowledge that AWS CloudFormation might create IAM resources with custom names</strong>” and choose <strong>Next</strong> <p><img class="alignnone size-full wp-image-86106" height="1073" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-5.png" width="1136" /></p></li> 
 <li>Review all the configuration details on the next page, then choose <strong>Submit</strong>.<br /> <img class="alignnone size-full wp-image-86105" height="625" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-6.png" width="1442" /></li> 
 <li>Wait for the stack creation to complete in the AWS CloudFormation console. The process typically takes approximately 35 to 40 minutes to provision all required resources.<br /> <img class="alignnone size-full wp-image-86104" height="547" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-7.png" width="1688" /><p></p> <p>The following table shows resources available in the AWS Account after CloudFormation template deployment is successfully completed:</p> 
  <table border="1px" cellpadding="10px" class="styled-table"> 
   <tbody> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Resource Type</strong></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Example Resource Name</strong></td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">S3 Buckets</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">For storing raw, processed data and assets</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834-mwaa-bucket-&lt;AWS_ACCOUNT&gt;-&lt;AWS_REGION&gt;,bdb4834-raw-bucket-&lt;AWS_ACCOUNT&gt;-&lt;AWS_REGION&gt;,bdb4834-formatted-bucket-&lt;AWS_ACCOUNT&gt;-&lt;AWS_REGION&gt;,bdb4834-published-bucket-&lt;AWS_ACCOUNT&gt;-&lt;AWS_REGION&gt;</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">IAM Role</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Role assumed by MWAA for permissions</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834-mwaa-role</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">MWAA Environment</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Managed Airflow environment for orchestration</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834-MyMWAACluster</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">VPC</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Network setup required by MWAA</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834-MyVPC</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Glue Catalog Databases</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Logical grouping of metadata for tables</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834_formatted_stg,bdb4834_formatted_exception, bdb4834_formatted, bdb4834_published</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Glue Crawlers</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Automatically catalog metadata from S3</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834-formatted-stg-crawler</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Lambda</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Lambda to Trigger MWAA DAG on file arrival and to setup Lake Formation Permissions</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">bdb4834_mwaa_trigger_process_s3_files,bdb4834-lf-tags-automation</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Lake Formation Setup</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Centralized governance and permissions</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">LF-Setup for the above Resources</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Airflow DAGs</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Airflow DAGs are stored in the S3 bucket named mwaa-bucket-&lt;AWS_ACCOUNT&gt;-&lt;AWS_REGION&gt; under the dags/ prefix. These DAGs are responsible for triggering data pipelines based on either file arrival events or scheduled intervals. The exact functionality of each DAG is explained in the following sections.</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">blog-test-data-processingcrawler-daily-runcreate-audit-tableprocess_raw_to_formatted_stage</td> 
    </tr> 
   </tbody> 
  </table> </li> 
 <li>When the stack is complete perform the below steps: 
  <ol type="a"> 
   <li>Open the <a href="https://us-east-1.console.aws.amazon.com/mwaa/home" rel="noopener noreferrer" target="_blank">Amazon Managed Workflows for Apache Airflow (MWAA)</a> console, choose on <strong>Open Airflow UI</strong></li> 
   <li>In the DAGs console, locate the following DAGs and unpause them by unchecking the toggle switch (radio button) next to each DAG.</li> 
  </ol> </li> 
</ol> 
<p><img class="alignnone size-full wp-image-86103" height="431" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-8.png" width="723" /></p> 
<h2>Add sample data to raw S3 bucket and create catalog tables</h2> 
<p>In this section, we upload sample data to raw S3 bucket (bucket name starting with bdb4834-raw-bucket) and convert the file formats to parquet and run <a href="https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue crawler</a> to create catalog tables that are used by dbt in the ELT Process. Glue Crawler automatically scans the data in S3 and creates or updates tables in the Glue Data Catalog, making the data queryable and accessible for transformation.</p> 
<ol> 
 <li>Download the <a href="https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/BDB-4834/sample_data.zip" rel="noopener noreferrer" target="_blank">sample</a> data.</li> 
 <li>Zip folder contains two sample data files, cards.json and customers.json<br /> <strong>Schema for cards.json</strong><p></p> 
  <table border="1px" cellpadding="10px" class="styled-table"> 
   <tbody> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Field</strong></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Data Type</strong></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">cust_id</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Unique customer identifier</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">cc_number</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Credit card number</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">cc_expiry_date</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Credit card expiry date</td> 
    </tr> 
   </tbody> 
  </table> <p><strong>Schema for customers.json</strong></p> 
  <table border="1px" cellpadding="10px" class="styled-table"> 
   <tbody> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Field</strong></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Data Type</strong></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">cust_id</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Unique customer identifier</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">fname</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">First name</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">lname</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Last name</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">gender</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Gender</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">address</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Full address</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">dob</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Date of birth (YYYY/MM/DD)</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">phone</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Phone number</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">email</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">String</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Email address</td> 
    </tr> 
   </tbody> 
  </table> </li> 
 <li>Open S3 console, choose <strong>General purpose buckets </strong>in the navigation pane.</li> 
 <li>Locate the S3 bucket with a name starting with bdb4834-raw-bucket. This bucket is created by the CloudFormation stack and can also be found under the stack’s <strong>Resources</strong> tab in the CloudFormation console.<img class="alignnone size-full wp-image-86102" height="313" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-9.png" width="854" /></li> 
 <li>Choose the bucket name to open it, and follow these steps to create the required prefix: 
  <ol type="a"> 
   <li>Choose <strong>Create folder</strong>.</li> 
   <li>Enter the folder name as mwaa/blog/partition_dt=YYYY-MM-DD/, replacing YYYY-MM-DD with the actual date to be used for the partition.</li> 
   <li>Choose <strong>Create folder</strong> to confirm.</li> 
  </ol> </li> 
 <li>Upload the sample data files from the location to the s3 raw bucket prefix.<br /> <img class="alignnone size-full wp-image-86101" height="391" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-10.png" width="920" /></li> 
 <li>As soon as the files are uploaded, the on_put object event on the raw bucket invokes <code>thebdb4834_mwaa_trigger_process_s3_files</code> lambda which triggers the process_raw_to_formatted_stg MWAA DAG. 
  <ol type="a"> 
   <li>In the Airflow UI, choose the process_raw_to_formatted_stg DAG to view execution status. This DAG converts the file formats to parquet and typically completes within a few seconds.<br /> <img class="alignnone size-full wp-image-86100" height="490" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-11.png" width="696" /></li> 
   <li>(Optional) To check the Lambda execution details: 
    <ol type="i"> 
     <li>On the <a href="https://console.aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda Console</a>, choose <strong>Functions</strong> in the navigation pane.</li> 
     <li>Select the function named bdb4834_mwaa_trigger_process_s3_files.<br /> <img class="alignnone size-full wp-image-86099" height="242" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-12.png" width="1009" /></li> 
    </ol> </li> 
  </ol> </li> 
 <li>Validate the parquet files are created in formatted bucket (bucket name starting with <code>bdb4834-formatted</code>) under the respective data object prefix.<br /> <img class="alignnone size-full wp-image-86098" height="398" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-13.png" width="918" /></li> 
 <li>Before proceeding further, re-upload the Lake Formation metadata file in MWAA bucket. 
  <ol type="a"> 
   <li>Open the S3 console, choose <strong>General purpose buckets </strong>in the navigation pane.</li> 
   <li>Search for the bucket starting with <strong>bdb4834-mwaa-bucket</strong><br /> <img class="alignnone size-full wp-image-86097" height="252" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-14.png" width="858" /></li> 
   <li>Choose the bucket name and go to the lakeformation prefix. Download the file named <code>lf_tags_metadata.json</code>. Now, re-upload the same file to the same location.<br /> <strong>Note:</strong> This re-upload is necessary because the Lambda function is configured to trigger on file arrival. When the resources were initially created by the CloudFormation stack, the files were simply moved to S3 and did not trigger the Lambda. Re-uploading the file ensures the Lambda function is executed as intended.</li> 
   <li>As soon as the file is uploaded, the on_put object event on the MWAA bucket invokes the <strong>lf_tags_automation</strong> lambda, which creates the Lake Formation (LF) tags as defined in the metadata file and grants access to the specified <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> roles for read/write.</li> 
   <li>Validate that the LF-Tags have been created by visiting the <a href="https://us-east-1.console.aws.amazon.com/lakeformation/home?region=us-east-1" rel="noopener noreferrer" target="_blank">Lake Formation Console</a>. In the left navigation pane, choose <strong>Permissions</strong>, and then select <strong>LF-Tags and permissions</strong>.<br /> <img class="alignnone size-full wp-image-86096" height="461" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-15.png" width="1365" /></li> 
  </ol> </li> 
 <li>Now, run the crawler DAG to create/update the catalog tables: <code>crawler-daily-run</code> 
  <ol type="a"> 
   <li>In the Airflow UI select the <code>crawler-daily-run</code> DAG and choose <strong>Trigger DAG</strong> to execute it.</li> 
   <li>This DAG is configured to trigger Glue Crawler which crawls the <code>formatted_stg</code> prefix under the <code>bdb4834-formatted</code> s3 bucket to create catalog tables as per the prefixes available under the <code>formatted_stg</code> prefix. 
    <div class="hide-language"> 
     <pre><code class="lang-code">bdb4834-formatted-bucket-&lt;aws-account-id&gt;-&lt;region&gt;/formatted_stg/
</code></pre> 
    </div> <p><img class="alignnone size-full wp-image-86095" height="308" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-16.png" width="1472" /></p></li> 
   <li>Monitor the execution of the crawler-daily-run DAG until it completes, which typically takes 2 to 3 minutes. The crawler run status can be verified in the AWS Glue Console by following these steps: 
    <ol type="i"> 
     <li>Open the <a href="https://us-east-1.console.aws.amazon.com/glue/home" rel="noopener noreferrer" target="_blank">AWS Glue Console</a>.</li> 
     <li>In the left navigation pane, choose <strong>Crawlers</strong>.</li> 
     <li>Search for the crawler named <code>bdb4834-formatted-stg-crawler</code>.</li> 
     <li>Check the <strong>Last run status</strong> column to confirm the crawler executed successfully.</li> 
     <li>Choose the crawler name to view additional run details and logs if needed.</li> 
    </ol> <p><img class="alignnone size-full wp-image-86094" height="635" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-17.png" width="1344" /></p></li> 
   <li>Once the crawler has completed successfully, in the left-hand panel, choose <strong>Databases</strong> and select the bdb4834_formatted_stg database to view the created tables, which should appear as showing in the following image. Optionally, select the table’s name to view its schema, and then select <strong>Table data</strong> to open Athena for data analysis. (An error may appear when querying data using Athena due to Lake Formation permissions. Review the Governance using Lake Formation section in this post to resolve the issue.)</li> 
  </ol> </li> 
</ol> 
<p><strong>Note:</strong> If this is the first time Athena is being used, a query result location must be configured by specifying an S3 bucket. Follow the instructions in the <a href="https://docs.aws.amazon.com/athena/latest/ug/query-results-specify-location-console.html" rel="noopener noreferrer" target="_blank">AWS Athena documentation</a> to set up the <strong>S3 staging bucket</strong> for storing query results.</p> 
<p><img class="alignnone size-full wp-image-86093" height="486" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-18.png" width="1653" /></p> 
<h2>Run model through DAG in MWAA</h2> 
<p>In this section, we cover how dbt models run in MWAA using <a href="https://aws.amazon.com/blogs/big-data/from-data-lakes-to-insights-dbt-adapter-for-amazon-athena-now-supported-in-dbt-cloud/" rel="noopener noreferrer" target="_blank">Athena adapter</a> to create Glue-catalogued tables and how auditing is done for each run.</p> 
<ol> 
 <li>After creating the tables in the Glue database using the AWS Glue Crawler in the previous steps, we can now proceed to run the dbt models in MWAA. These models are stored in S3 in the form of SQL files, located at the S3 prefix<strong>: </strong>bdb4834-mwaa-bucket-&lt;account_id&gt;-us-east-1/dags/dbt/models/<br /> The following are the dbt models and their functionality:<p></p> 
  <ul> 
   <li><code>mwaa_blog_cards_exception.sql</code> This model reads data from the <code>mwaa_blog_cards</code> table in the <code>bdb4834_formatted_stg</code> database and writes records with data quality issues to the <code>mwaa_blog_cards_exception</code> table in the <code>bdb4834_formatted_exception</code> database.</li> 
   <li><code>mwaa_blog_customers_exception.sql</code> This model reads data from the <code>mwaa_blog_customers</code> table in the <code>bdb4834_formatted_stg</code> database and writes records with data quality issues to the <code>mwaa_blog_customers_exception</code> table in the <code>bdb4834_formatted_exception</code> database.</li> 
   <li><code>mwaa_blog_cards.sql</code> This model reads data from the <code>mwaa_blog_cards</code> table in the <code>bdb4834_formatted_stg</code> database and loads it into the <code>mwaa_blog_cards</code> table in the <code>bdb4834_formatted</code> database. If the target table does not exist, dbt automatically creates it.</li> 
   <li><code>mwaa_blog_customers.sql</code> This model reads data from the <code>mwaa_blog_customers</code> table in the <code>bdb4834_formatted_stg</code> database and loads it into the <code>mwaa_blog_customers</code> table in the <code>bdb4834_formatted</code> database. If the target table does not exist, dbt automatically creates it.</li> 
  </ul> </li> 
 <li>The <code>mwaa_blog_cards.sql</code> model processes credit card data and depends on the <code>mwaa_blog_customers.sql</code> model to complete successfully before it runs. This dependency is necessary because certain data quality checks—such as referential integrity validations between customer and card records—must be performed beforehand. 
  <ul> 
   <li>These relationships and checks are defined in the <code>schema.yml</code> file located in the same S3 path: <code>bdb4834-mwaa-bucket-&lt;account_id&gt;-us-east-1/dags/dbt/models/</code>. The schema.yml file provides metadata for dbt models, including model dependencies, column definitions, and data quality tests. It utilizes macros like <code>get_dq_macro.sql</code> and <code>dq_referentialcheck.sql</code> (found under the macros/ directory) to enforce these validations.</li> 
  </ul> <p> As a result, dbt automatically generates a <strong>lineage graph</strong> based on the declared dependencies. This visual graph helps orchestrate model execution order—ensuring models like <code>mwaa_blog_customers.sql</code> run before dependent models such as <code>mwaa_blog_cards.sql</code>, and identifies which models can execute in parallel to optimize the pipeline.</p></li> 
 <li>As a pre-step before running models, choose the trigger DAG button for create-audit-table to create audit table for storing run details for each model.<br /> <img class="alignnone size-full wp-image-86092" height="89" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-19.png" width="1378" /></li> 
 <li>Trigger the <code>blog-test-data-processing</code> DAG in the Airflow UI to start the Model run.</li> 
 <li>Choose <code>blog-test-data-processing</code> to see the execution status. This DAG runs the models in order and creates Glue catalogued iceberg tables. The flow diagram of a DAG from Airflow UI can be found by choosing <strong>Graph</strong> after choosing <strong>DAG</strong>.<br /> <img class="alignnone size-full wp-image-86091" height="454" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-20.png" width="1377" /><br /> <img class="alignnone size-full wp-image-86090" height="480" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-21.png" width="1326" /><p></p> 
  <ol type="a"> 
   <li>The exception models puts the failed records under exception prefix in S3: 
    <div class="hide-language"> 
     <pre><code class="lang-code">bdb4834-formatted-bucket-&lt;aws-account-id&gt;-&lt;region&gt;/formatted_exception/</code></pre> 
    </div> <p> Records that failed are found in an added column, <strong>tests_failed</strong>, where all the data quality checks that failed for that particular row are added, separated by a pipe (‘|’). (For the <code>mwaa_blog_customers_exception</code> two exception records are found in the table.)</p></li> 
   <li>The passed records are put under formatted prefix in S3. 
    <div class="hide-language"> 
     <pre><code class="lang-code">bdb4834-formatted-bucket-&lt;aws-account-id&gt;-&lt;region&gt;/formatted/</code></pre> 
    </div> </li> 
   <li>For each run, a run audit is captured in the audit table with execution details like model_nm, process_nm, execution_start_date, execution_end_date, execution_status, execution_failure_reason, rows_affected.<br /> Find the data in S3 under the prefix <code>bdb4834-formatted-bucket-&lt;aws-account-id&gt;-&lt;region&gt;/audit_control/</code></li> 
   <li>Monitor the execution until the DAG completes, which can take up to 2-3 mins. The execution status of the DAG can be seen in the left panel after opening the DAG.<br /> <img class="alignnone size-full wp-image-86089" height="511" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-22.png" width="1070" /></li> 
   <li>Once the DAG has completed successfully, open the <a href="https://us-east-1.console.aws.amazon.com/glue/home?region=us-east-1#/v2/getting-started" rel="noopener noreferrer" target="_blank">AWS Glue console</a> and select <strong>Databases</strong>. Select the <code>bdb4834_formatted</code> database, which should create three tables, as shown in the following image.<br /> Optionally, choose <strong>Table data</strong> to access Athena for data analysis.<br /> <img class="alignnone size-full wp-image-86088" height="504" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-23.png" width="1646" /></li> 
   <li>Choose <code>bdb4834_formatted_exception</code> database from under <strong>Databases</strong> in AWS Glue console, which should create two tables as shown in the following image.<img class="alignnone size-full wp-image-86087" height="469" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-24.png" width="1490" /></li> 
   <li>Each model is assigned LF tags through the config block of model itself. Therefore, when the iceberg tables are created through dbt, LF tags are attached to the tables after the run completes. <p> Validate the LF tags attached to the tables by visiting the AWS Lake Formation console. In the left navigation pane, choose <strong>Tables</strong> and look for <code>mwaa_blog_customers</code> or <code>mwaa_blog_cards</code> table under <code>bdb4834_formatted</code> database. Select any table among the two and under <strong>Actions</strong>, choose <strong>Edit LF tags</strong> and the tags are attached, as shown in the following screen shot.<br /> <img class="alignnone size-full wp-image-86086" height="429" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-25.png" width="798" /><br /> <img class="alignnone size-full wp-image-86085" height="436" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-26.png" width="801" /></p></li> 
   <li>Similarly, for the <code>bdb4834_formatted_exception</code> database, select any one of the exception tables under the <code>bdb4834_formatted_exception</code> database and the LF tags are attached.<br /> <img class="alignnone size-full wp-image-86084" height="478" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-27.png" width="798" /></li> 
   <li>Run SQL queries on the tables created by opening the Athena console and running <strong>Analytical queries</strong> on the tables created above.Sample SQL queries: 
    <div class="hide-language"> 
     <pre><code class="lang-sql">SELECT * FROM bdb4834_formatted.mwaa_blog_cards;
Output: Total 30 rows</code></pre> 
    </div> <p> <img class="alignnone size-full wp-image-86128" height="285" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-28.png" width="451" /></p> 
    <div class="hide-language"> 
     <pre><code class="lang-sql">SELECT * FROM bdb4834_formatted_exception.mwaa_blog_customers_exception;
Output: Total 2 records</code></pre> 
    </div> <p> <img class="alignnone size-full wp-image-86127" height="21" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-29.png" width="451" /></p></li> 
  </ol> </li> 
</ol> 
<h2>Governance using Lake Formation</h2> 
<p>In this section, we show how assigning Lake Formation permissions and creating LF tags is automated using the metadata file.Below is a metadata file structure, which is needed for reference when uploading the metadata file for Lake Formation in Airflow S3 bucket, inside the Lake Formation prefix.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">Metadata file structure-
{
    "role_arn": "&lt;&lt;IAM_ROLE_ARN&gt;&gt;",
    "access_type": "GRANT",
    "lf_tags": [
      {
        "TagKey": "&lt;&lt;LF_tag_key&gt;&gt;",
        "TagValues": ["&lt;&lt;LF_tag_values&gt;&gt;"]
      }
    ],
	  "named_data_catalog": [
      {
        "Database": "&lt;&lt;Database_Name&gt;&gt;",
        "Table": ""&lt;&lt;Table_Name&gt;&gt;"
      }
    ],
    "table_permissions": ["SELECT", "DESCRIBE"]
  }
</code></pre> 
</div> 
<p>Components of the metadata file</p> 
<ul> 
 <li><code>role_arn</code>: The IAM role that the Lambda function assumes to perform operations.</li> 
 <li><code>access_type</code>: Specifies whether the action is to grant or revoke permissions (GRANT, REVOKE).</li> 
 <li><code>lf_tags</code>: Tags used for tag-based access control (TBAC) in Lake Formation.</li> 
 <li><code>named_data_catalog</code>: A list of databases and tables on which Lake Formation permissions or tags are applied to.</li> 
 <li><code>table_permissions</code>: Lake Formation-specific permissions (e.g., SELECT, DESCRIBE, ALTER, etc.).</li> 
</ul> 
<p>Lambda function <code>bdb4834-lf-tags-automation</code> parses this JSON and grants the required LF tags to the role with given table permissions.</p> 
<ol> 
 <li>To update the metadata file, download it from the MWAA bucket (lakeformation prefix) 
  <div class="hide-language"> 
   <pre><code class="lang-code">bdb4834-mwaa-bucket-&lt;&lt;ACCOUNT_NO&gt;&gt;-&lt;&lt;REGION&gt;&gt;/lakeformation/lf_tags_metadata.json</code></pre> 
  </div> </li> 
 <li>Add a JSON object with the metadata structure defined above, mentioning the IAM role ARN and the tags and tables to which access needs to be granted.<br /> Example:Let’s assume below is how the metadata file initially looks like:<p></p> 
  <div class="hide-language"> 
   <pre><code class="lang-css">
	[
	{
    "role_arn": "arn:aws:iam::XXX:role/aws-reserved/sso.amazonaws.com/XX ",
    "access_type": "GRANT",
    "lf_tags": [
      {
        "TagKey": " blog",
        "TagValues": ["bdb-4834"]
      }
    ],
    "named_data_catalog": [],
    "table_permissions": ["SELECT", "DESCRIBE"]
  }
]</code></pre> 
  </div> <p>Below is the json object that has to be added in the above metadata file:</p> 
  <div class="hide-language"> 
   <pre><code class="lang-css">
{
          "role_arn": "arn:aws:iam::XXX:role/aws-reserved/sso.amazonaws.com/XX ",
          "access_type": "GRANT",
          "lf_tags": [],
          "named_data_catalog": [
          {
            "Database": " bdb4834_formatted",
            "Table": "audit_control"
          },
          {
            "Database": " bdb4834_formatted_stg",
            "Table": "*"
          }
         ],
         "table_permissions": ["SELECT", "DESCRIBE"]}


</code></pre> 
  </div> <p>So now, the final metadata file should look like:</p> 
  <div class="hide-language"> 
   <pre><code class="lang-css">
[
  {
    "role_arn": "arn:aws:iam::XXX:role/aws-reserved/sso.amazonaws.com/XX ",
    "access_type": "GRANT",
    "lf_tags": [
      {
        "TagKey": "blog",
        "TagValues": ["bdb-4834"]
      }
    ],
    "named_data_catalog": [],
    "table_permissions": ["SELECT", "DESCRIBE"]
  },
  {
    "role_arn": "arn:aws:iam::XXX:role/aws-reserved/sso.amazonaws.com/XX ",
    "access_type": "GRANT",
    "lf_tags": [],
    "named_data_catalog": [
      {
        "Database": " bdb4834_formatted",
        "Table": "audit_control"
      },
      {
        "Database": " bdb4834_formatted_stg",
        "Table": "*"
      }
    ],
    "table_permissions": ["SELECT", "DESCRIBE"]
  }
]</code></pre> 
  </div> </li> 
 <li>Upon uploading this file at the same location (<code>bdb4834-mwaa-bucket-&lt;&lt;ACCOUNT_NO&gt;&gt;-&lt;&lt;REGION&gt;&gt;/lakeformation/</code>) in S3, the <code>lf_tags_automation</code> lambda is triggered to create LF tags if they don’t exist and then it assigns those tags to the IAM role ARN and also grants permission to the IAM role ARN using <code>named_data_catalog</code> as defined. <p>To verify the permissions, go to the <a href="https://us-east-1.console.aws.amazon.com/lakeformation" rel="noopener noreferrer" target="_blank">Lake Formation console</a> and choose <strong>Tables</strong> under <strong>Data Catalog</strong> and search for the table name.<img class="alignnone size-full wp-image-86083" height="443" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-30.png" width="1157" /></p></li> 
</ol> 
<p>To check LF-Tags, choose the table name and under the <strong>LF tags</strong> section, all the tags are found attached to this table.<br /> <img class="alignnone size-full wp-image-86082" height="658" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-31.png" width="1377" /></p> 
<p>This metadata file used as a structured input to an <strong>AWS Lambda function</strong> automates the following to perform <strong>automated, consistent, and scalable data access governance</strong> across the AWS Lake Formation environments:</p> 
<ul> 
 <li><strong>Granting AWS Lake Formation (LF) permissions</strong> on Glue Data Catalog resources (like databases and tables).</li> 
 <li><strong>Creating Lake Formation Tags and Applying Lake Formation tags (LF-Tags)</strong> for tag-based access control (TBAC).</li> 
</ul> 
<h2>Explore more on dbt</h2> 
<p>Now that the deployment includes a bdb4834-<strong>published S3 bucket</strong> and a <strong>published Catalog database,</strong> robust dbt models can be built for data transformation and curation.</p> 
<p>Here’s how to implement a complete dbt workflow:</p> 
<ul> 
 <li>Start by developing models that follow this pattern: 
  <ul> 
   <li>Read from the formatted tables in the staging area</li> 
   <li>Apply business logic, joins, and aggregations</li> 
   <li>Write clean, analysis-ready data to the published schema</li> 
  </ul> </li> 
 <li><strong>Tagging for automation:</strong> Use consistent dbt tags to enable automatic DAG generation. These tags trigger MWAA orchestration to automatically include new models in the execution pipeline.</li> 
 <li><strong>Adding new models:</strong> When working with new datasets, refer to existing models for guidance. Apply appropriate LF tags for data access control. The new <strong>LF tags can also now be used</strong> for permissions.</li> 
 <li><strong>Enable DAG execution:</strong> For new datasets, update the <strong>MWAA metadata file</strong> to include a <strong>new JSON entry</strong>. This step is necessary to generate a DAG that executes the new dbt models.</li> 
</ul> 
<p>This approach ensures the dbt implementation scales systematically while maintaining automated orchestration and proper data governance.</p> 
<h2>Clean up</h2> 
<p>1. Open the S3 console and delete all objects from below buckets:</p> 
<ul> 
 <li>bdb4834-raw-bucket-&lt;aws-account-id&gt;-&lt;region&gt;</li> 
 <li>bdb4834-formatted -bucket-&lt;aws-account-id&gt;-&lt;region&gt;</li> 
 <li>bdb4834-mwaa-bucket-&lt;aws-account-id&gt;-&lt;region&gt;</li> 
 <li>bdb4834-published-bucket-&lt;aws-account-id&gt;-&lt;region&gt;</li> 
</ul> 
<p>To delete all objects, choose the bucket name, select all objects and choose <strong>Delete</strong>.</p> 
<p>After that, type ‘permanently delete’ in the text box and choose <strong>Delete Objects</strong>.</p> 
<p>Do this for all three buckets mentioned above.</p> 
<p><img class="alignnone size-full wp-image-86081" height="479" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-32.png" width="1748" /></p> 
<p><img class="alignnone size-full wp-image-86080" height="516" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-33.png" width="1261" /></p> 
<p>2. Go to the <a href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1" rel="noopener noreferrer" target="_blank">AWS Cloudformation console</a>, choose you’re the stack name and select <strong>Delete</strong>. It may take approximately 40 mins for the deletion to complete.</p> 
<h2>Recommendations</h2> 
<p>When using dbt with MWAA, some typical challenges include worker resource exhaustion, dependency management issues, and in some rare cases, issues like DAGs disappearing and re-appearing when there are a large number of dynamic DAGs being created from a single python script.</p> 
<p>To mitigate these issues, follow these best practices:</p> 
<p>1. Scale the MWAA environment appropriately by <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/environment-class.html" rel="noopener noreferrer" target="_blank">upgrading the environment class</a> as required.</p> 
<p>2. Use <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/working-dags-dependencies.html" rel="noopener noreferrer" target="_blank">custom requirements.txt and proper dbt adapter configuration</a> to ensure consistent environments.</p> 
<p>3. Set airflow configuration parameters to <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/best-practices-tuning.html" rel="noopener noreferrer" target="_blank">tune the performance of MWAA</a>.</p> 
<h2>Conclusion</h2> 
<p>In this post, we explored the <strong>end-to-end setup of a governed data lake</strong> using <strong>MWAA and dbt which </strong>improved data quality, security, and compliance, leading to better decision-making and increased operational efficiency. We also covered how to build <strong>custom dbt frameworks</strong> for auditing and data quality, automate <strong>Lake Formation access control</strong>, and dynamically generate <strong>MWAA DAGs</strong> based on dbt tags. These capabilities enable a scalable, secure, and automated data lake architecture, streamlining data governance and orchestration.</p> 
<p>For further exploring, refer to <a href="https://aws.amazon.com/blogs/big-data/from-data-lakes-to-insights-dbt-adapter-for-amazon-athena-now-supported-in-dbt-cloud/" rel="noopener noreferrer" target="_blank">From data lakes to insights: dbt adapter for Amazon Athena now supported in dbt Cloud</a></p> 
<hr /> 
<h3>About the authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Muralidhar Reddy" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/Murali-ID-pic.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Muralidhar Reddy</h3> 
  <p><a href="https://www.linkedin.com/in/muralidhar-reddy-ab59b741/" rel="noopener" target="_blank">Muralidhar</a> is a Delivery Consultant at Amazon Web Services (AWS), helping customers build and implement data analytics solution. When he’s not working, Murali is an avid bike rider and loves exploring new places.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Abhilasha Agarwal" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/12/12/BDB-4834-image-35.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Abhilasha Agarwal</h3> 
  <p><a href="https://www.linkedin.com/in/abhilasha-agarwal-2501/" rel="noopener" target="_blank">Abhilasha</a> is an Associate Delivery Consultant at Amazon Web Services (AWS), support customers in building robust data analytics solutions. Apart from work, she loves cooking and trying out fun outdoor experiences.</p>
 </div> 
</footer>

---
title: "Break down data silos and seamlessly query Iceberg tables in Amazon SageMaker from Snowflake"
url: "https://aws.amazon.com/blogs/big-data/break-down-data-silos-and-seamlessly-query-iceberg-tables-in-amazon-sagemaker-from-snowflake/"
date: "Mon, 15 Sep 2025 20:12:22 +0000"
author: "Nidhi Gupta"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p>Organizations often struggle to unify their data ecosystems across multiple platforms and services. The connectivity between <a href="https://aws.amazon.com/sagemaker/" rel="noopener noreferrer" target="_blank">Amazon SageMaker</a> and <a href="https://www.snowflake.com/en/" rel="noopener noreferrer" target="_blank">Snowflake’s AI Data Cloud</a> offers a powerful solution to this challenge, so businesses can take advantage of the strengths of both environments while maintaining a cohesive data strategy.</p> 
<p>In this post, we demonstrate how you can break down data silos and enhance your analytical capabilities by querying Apache Iceberg tables in the <a href="https://aws.amazon.com/sagemaker/lakehouse/" rel="noopener noreferrer" target="_blank">lakehouse architecture of SageMaker</a> directly from Snowflake. With this capability, you can access and analyze data stored in <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) through <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a> using an <a href="https://docs.aws.amazon.com/glue/latest/dg/access_catalog.html" rel="noopener noreferrer" target="_blank">AWS Glue Iceberg REST endpoint</a>, all secured by <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a>, without the need for complex extract, transform, and load (ETL) processes or data duplication. You can also automate table discovery and refresh using <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database" rel="noopener noreferrer" target="_blank">Snowflake catalog-linked databases for Iceberg</a>. In the following sections, we show how to set up this integration so Snowflake users can seamlessly query and analyze data stored in AWS, thereby improving data accessibility, reducing redundancy, and enabling more comprehensive analytics across your entire data ecosystem.</p> 
<h2>Business use cases and key benefits</h2> 
<p>The capability to query Iceberg tables in SageMaker from Snowflake delivers significant value across multiple industries:</p> 
<ul> 
 <li><strong>Financial services</strong> – Enhance fraud detection through unified analysis of transaction data and customer behavior patterns</li> 
 <li><strong>Healthcare</strong> – Improve patient outcomes through integrated access to clinical, claims, and research data</li> 
 <li><strong>Retail</strong> – Increase customer retention rates by connecting sales, inventory, and customer behavior data for personalized experiences</li> 
 <li><strong>Manufacturing</strong> – Boost production efficiency through unified sensor and operational data analytics</li> 
 <li><strong>Telecommunications</strong> – Reduce customer churn with comprehensive analysis of network performance and customer usage data</li> 
</ul> 
<p>Key benefits of this capability include:</p> 
<ul> 
 <li><strong>Accelerated decision-making</strong> – Reduce time to insight through integrated data access across platforms</li> 
 <li><strong>Cost optimization</strong> – Accelerate time to insight by querying data directly in storage without the need for ingestion</li> 
 <li><strong>Improved data fidelity</strong> – Reduce data inconsistencies by establishing a single source of truth</li> 
 <li><strong>Enhanced collaboration</strong> – Increase cross-functional productivity through simplified data sharing between data scientists and analysts</li> 
</ul> 
<p>By using the lakehouse architecture of SageMaker with Snowflake’s serverless and zero-tuning computational power, you can break down data silos, enabling comprehensive analytics and democratizing data access. This integration supports a modern data architecture that prioritizes flexibility, security, and analytical performance, ultimately driving faster, more informed decision-making across the enterprise.</p> 
<h2>Solution overview</h2> 
<p>The following diagram shows the architecture for catalog integration between Snowflake and Iceberg tables in the lakehouse.</p> 
<p><img alt="Catalog integration to query Iceberg tables in S3 bucket using Iceberg REST Catalog (IRC) with credential vending" class="alignnone wp-image-83125 size-full" height="1216" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/09/05/image-1-6.png" width="2372" /></p> 
<p>The workflow consists of the following components:</p> 
<ul> 
 <li>Data storage and management: 
  <ul> 
   <li>Amazon S3 serves as the primary storage layer, hosting the Iceberg table data</li> 
   <li>The Data Catalog maintains the metadata for these tables</li> 
   <li>Lake Formation provides credential vending</li> 
  </ul> </li> 
 <li>Authentication flow: 
  <ul> 
   <li>Snowflake initiates queries using a catalog integration configuration</li> 
   <li>Lake Formation vends temporary credentials through <a href="https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html" rel="noopener noreferrer" target="_blank">AWS Security Token Service</a> (AWS STS)</li> 
   <li>These credentials are automatically refreshed based on the configured refresh interval</li> 
  </ul> </li> 
 <li>Query flow: 
  <ul> 
   <li>Snowflake users submit queries against the mounted Iceberg tables</li> 
   <li>The AWS Glue Iceberg REST endpoint processes these requests</li> 
   <li>Query execution uses Snowflake’s compute resources while reading directly from Amazon S3</li> 
   <li>Results are returned to Snowflake users while maintaining all security controls</li> 
  </ul> </li> 
</ul> 
<p>There are four patterns to query Iceberg tables in SageMaker from Snowflake:</p> 
<ul> 
 <li>Iceberg tables in an S3 bucket using an AWS Glue Iceberg REST endpoint and Snowflake Iceberg REST catalog integration, with credential vending from Lake Formation</li> 
 <li>Iceberg tables in an S3 bucket using an AWS Glue Iceberg REST endpoint and Snowflake Iceberg REST catalog integration, using Snowflake external volumes to Amazon S3 data storage</li> 
 <li>Iceberg tables in an S3 bucket using AWS Glue API catalog integration, also using Snowflake external volumes to Amazon S3</li> 
 <li><a href="https://aws.amazon.com/blogs/storage/connect-snowflake-to-s3-tables-using-the-sagemaker-lakehouse-iceberg-rest-endpoint/" rel="noopener noreferrer" target="_blank">Amazon S3 Tables using Iceberg REST catalog integration with credential vending</a> from Lake Formation</li> 
</ul> 
<p>In this post, we implement the first of these four access patterns using <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest#sigv4-glue" rel="noopener noreferrer" target="_blank">catalog integration</a> for the AWS Glue Iceberg REST endpoint with <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html" rel="noopener noreferrer" target="_blank">Signature Version 4 (SigV4)</a> authentication in Snowflake.</p> 
<h2>Prerequisites</h2> 
<p>You must have the following prerequisites:</p> 
<ul> 
 <li>A <a href="https://signup.snowflake.com/" rel="noopener noreferrer" target="_blank">Snowflake account</a>.</li> 
 <li>An <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) role that is a Lake Formation data lake administrator in your AWS account. A data lake administrator is an IAM principal that can register Amazon S3 locations, access the Data Catalog, grant Lake Formation permissions to other users, and view <a href="https://aws.amazon.com/cloudtrail" rel="noopener noreferrer" target="_blank">AWS CloudTrail</a>. See <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/initial-LF-setup.html#create-data-lake-admin" rel="noopener noreferrer" target="_blank">Create a data lake administrator</a> for more information.</li> 
 <li>An existing <a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a> database named <code>iceberg_db</code> and Iceberg table named <code>customer</code> with data stored in an S3 general purpose bucket with a unique name. To create the table, refer to the <a href="https://github.com/gregrahn/tpch-kit/blob/master/dbgen/dss.ddl" rel="noopener noreferrer" target="_blank">table schema</a> and <a href="https://github.com/gregrahn/tpch-kit/blob/master/ref_data/1/customer.tbl.1" rel="noopener noreferrer" target="_blank">dataset</a>.</li> 
 <li>A user-defined IAM role that Lake Formation assumes when accessing the data in the aforementioned S3 location to vend scoped credentials (see <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/registration-role.html" rel="noopener noreferrer" target="_blank">Requirements for roles used to register locations</a>). For this post, we use the IAM role <code>LakeFormationLocationRegistrationRole</code>.</li> 
</ul> 
<p>The solution takes approximately 30–45 minutes to set up. Cost varies based on data volume and query frequency. Use the <a href="https://calculator.aws/#/" rel="noopener noreferrer" target="_blank">AWS Pricing Calculator</a> for specific estimates.</p> 
<h2>Create an IAM role for Snowflake</h2> 
<p>To create an IAM role for Snowflake, you first create a policy for the role:</p> 
<ol> 
 <li>On the IAM console, choose <strong>Policies</strong> in the navigation pane.</li> 
 <li>Choose <strong>Create policy</strong>.</li> 
 <li>Choose the JSON editor and enter the following policy (provide your AWS Region and account ID), then choose <strong>Next</strong>.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
 &nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
 &nbsp;&nbsp; &nbsp;"Statement": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "AllowGlueCatalogTableAccess",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetCatalog",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetCatalogs",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetPartitions",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetPartition",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetDatabase",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetDatabases",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetTable",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetTables",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:UpdateTable"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:glue:&lt;region&gt;:&lt;account-id&gt;:catalog",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:glue:&lt;region&gt;:&lt;account-id&gt;:database/iceberg_db",
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "arn:aws:glue:&lt;region&gt;:&lt;account-id&gt;:table/iceberg_db/*",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"lakeformation:GetDataAccess"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
 &nbsp;&nbsp; &nbsp;]
 }</code></pre> 
</div> 
<ol start="4"> 
 <li>Enter <code>iceberg-table-access</code> as the policy name.</li> 
 <li>Choose <strong>Create policy</strong>.</li> 
</ol> 
<p>Now you can create the role and attach the policy you created.</p> 
<ol start="6"> 
 <li>Choose <strong>Roles</strong> in the navigation pane.</li> 
 <li>Choose <strong>Create role</strong>.</li> 
 <li>Choose <strong>AWS account</strong>.</li> 
 <li>Under <strong>Options</strong>, select <strong>Require External Id</strong> and enter an external ID of your choice.</li> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>Choose the policy you created (<code>iceberg-table-access policy</code>).</li> 
 <li>Enter <code>snowflake_access_role</code> as the role name.</li> 
 <li>Choose <strong>Create role</strong>.</li> 
</ol> 
<h2>Configure Lake Formation access controls</h2> 
<p>To configure your Lake Formation access controls, first set up the application integration:</p> 
<ol> 
 <li>Sign in to the Lake Formation console as a data lake administrator.</li> 
 <li>Choose <strong>Administration</strong> in the navigation pane.</li> 
 <li>Select <strong>Application integration settings</strong>.</li> 
 <li>Enable <strong>Allow external engines to access data in Amazon S3 locations with full table access</strong>.</li> 
 <li>Choose <strong>Save</strong>.</li> 
</ol> 
<p>Now you can grant permissions to the IAM role.</p> 
<ol start="6"> 
 <li>Choose <strong>Data permissions</strong> in the navigation pane.</li> 
 <li>Choose <strong>Grant</strong>.</li> 
 <li>Configure the following settings: 
  <ol type="a"> 
   <li>For <strong>Principals</strong>, select <strong>IAM users and roles</strong> and choose <code>snowflake_access_role</code>.</li> 
   <li>For <strong>Resources</strong>, select <strong>Named Data Catalog resources</strong>.</li> 
   <li>For <strong>Catalog</strong>, choose your AWS account ID.</li> 
   <li>For <strong>Database</strong>, choose <code>iceberg_db</code>.</li> 
   <li>For <strong>Table</strong>, choose <code>customer</code>.</li> 
   <li>For <strong>Permissions</strong>, select <strong>SUPER</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Grant</strong>.</li> 
</ol> 
<p>SUPER access is required for mounting the Iceberg table in Amazon S3 as a Snowflake table.</p> 
<h2>Register the S3 data lake location</h2> 
<p>Complete the following steps to register the S3 data lake location:</p> 
<ol> 
 <li>As data lake administrator on the Lake Formation console, choose <strong>Data lake locations</strong> in the navigation pane.</li> 
 <li>Choose <strong>Register location</strong>.</li> 
 <li>Configure the following: 
  <ol type="a"> 
   <li>For <strong>S3 path</strong>, enter the S3 path to the bucket where you will store your data.</li> 
   <li>For <strong>IAM role</strong>, choose <code>LakeFormationLocationRegistrationRole</code>.</li> 
   <li>For <strong>Permission mode</strong>, choose <strong>Lake Formation</strong>.</li> 
  </ol> </li> 
 <li>Choose <strong>Register location</strong>.</li> 
</ol> 
<h2>Set up the Iceberg REST integration in Snowflake</h2> 
<p>Complete the following steps to set up the Iceberg REST integration in Snowflake:</p> 
<ol> 
 <li>Log in to Snowflake as an admin user.</li> 
 <li>Execute the following SQL command (provide your Region, account ID, and external ID that you provided during IAM role creation):</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-sql">CREATE OR REPLACE CATALOG INTEGRATION glue_irc_catalog_int
CATALOG_SOURCE = ICEBERG_REST
TABLE_FORMAT = ICEBERG
CATALOG_NAMESPACE = 'iceberg_db'
REST_CONFIG = (
&nbsp;&nbsp; &nbsp;CATALOG_URI = 'https://glue.&lt;region&gt;.amazonaws.com/iceberg'
&nbsp;&nbsp; &nbsp;CATALOG_API_TYPE = AWS_GLUE
&nbsp;&nbsp; &nbsp;CATALOG_NAME = '&lt;account-id&gt;'
&nbsp;&nbsp; &nbsp;ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
)
REST_AUTHENTICATION = (
&nbsp;&nbsp; &nbsp;TYPE = SIGV4
&nbsp;&nbsp; &nbsp;SIGV4_IAM_ROLE = 'arn:aws:iam::&lt;account-id&gt;:role/snowflake_access_role'
&nbsp;&nbsp; &nbsp;SIGV4_SIGNING_REGION = '&lt;region&gt;'
    SIGV4_EXTERNAL_ID = '&lt;external-id&gt;'
)
REFRESH_INTERVAL_SECONDS = 120
ENABLED = TRUE;</code></pre> 
</div> 
<ol start="3"> 
 <li>Execute the following SQL command and retrieve the value for <code>API_AWS_IAM_USER_ARN</code>:</li> 
</ol> 
<p><code>DESCRIBE CATALOG INTEGRATION glue_irc_catalog_int;</code></p> 
<ol start="4"> 
 <li>On the IAM console, update the trust relationship for <code>snowflake_access_role</code> with the value for <code>API_AWS_IAM_USER_ARN</code>:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
&nbsp; &nbsp; "Version": "2012-10-17",
&nbsp;&nbsp;&nbsp; "Statement": [
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Sid": "",
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Effect": "Allow",
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Principal": {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "AWS": [
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"&lt;API_AWS_IAM_USER_ARN&gt;"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ]
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; },
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Action": "sts:AssumeRole",
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Condition": {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "StringEquals": {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "sts:ExternalId": [
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "&lt;external-id&gt;"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ]
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp;&nbsp;&nbsp; ]
}</code></pre> 
</div> 
<ol start="5"> 
 <li>Verify the catalog integration:</li> 
</ol> 
<p><code>SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('glue_irc_catalog_int');</code></p> 
<ol start="6"> 
 <li>Mount the S3 table as a Snowflake table:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-sql">CREATE OR REPLACE ICEBERG TABLE s3iceberg_customer
 CATALOG = 'glue_irc_catalog_int'
 CATALOG_NAMESPACE = 'iceberg_db'
 CATALOG_TABLE_NAME = 'customer'
 AUTO_REFRESH = TRUE;</code></pre> 
</div> 
<h2>Query the Iceberg table from Snowflake</h2> 
<p>To test the configuration, log in to Snowflake as an admin user and run the following sample query:<code>SELECT * FROM s3iceberg_customer LIMIT 10;</code></p> 
<h2>Clean up</h2> 
<p>To clean up your resources, complete the following steps:</p> 
<ol> 
 <li>Delete the database and table in AWS Glue.</li> 
 <li>Drop the Iceberg table, catalog integration, and database in Snowflake:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-sql">DROP ICEBERG TABLE iceberg_customer;
DROP CATALOG INTEGRATION glue_irc_catalog_int;</code></pre> 
</div> 
<p>Make sure all resources are properly cleaned up to avoid unexpected charges.</p> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to establish a secure and efficient connection between your Snowflake environment and SageMaker to query Iceberg tables in Amazon S3. This capability can help your organization maintain a single source of truth while also letting teams use their preferred analytics tools, ultimately breaking down data silos and enhancing collaborative analysis capabilities.</p> 
<p>To further explore and implement this solution in your environment, consider the following resources:</p> 
<ul> 
 <li>Technical documentation: 
  <ul> 
   <li>Review the <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/lakehouse.html" rel="noopener noreferrer" target="_blank">Amazon SageMaker Lakehouse User Guide</a></li> 
   <li>Explore <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/security.html" rel="noopener noreferrer" target="_blank">Security in AWS Lake Formation</a> for best practices to optimize your security controls</li> 
   <li>Learn more about <a href="https://iceberg.apache.org/" rel="noopener noreferrer" target="_blank">Iceberg table format</a> and its benefits for data lakes</li> 
   <li>Refer to <a href="https://docs.snowflake.com/en/user-guide/data-load-s3-config" rel="noopener noreferrer" target="_blank">Configuring secure access from Snowflake to Amazon S3</a></li> 
  </ul> </li> 
 <li>Related blog posts: 
  <ul> 
   <li><a href="https://aws.amazon.com/blogs/apn/build-real-time-data-lakes-with-snowflake-and-amazon-s3-tables/" rel="noopener noreferrer" target="_blank">Build real-time data lakes with Snowflake and Amazon S3 Tables</a></li> 
   <li><a href="https://aws.amazon.com/blogs/big-data/simplify-data-access-for-your-enterprise-using-amazon-sagemaker-lakehouse/" rel="noopener noreferrer" target="_blank">Simplify data access for your enterprise using Amazon SageMaker Lakehouse</a></li> 
  </ul> </li> 
</ul> 
<p>These resources can help you to implement and optimize this integration pattern for your specific use case. As you begin this journey, remember to start small, validate your architecture with test data, and gradually scale your implementation based on your organization’s needs.</p> 
<hr /> 
<h3>About the authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Nidhi Gupta" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/09/05/Nidhi-pic-100x101.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Nidhi Gupta</h3> 
  <p><a href="https://www.linkedin.com/in/nidhi-gupta-5b80874/" rel="noopener" target="_blank">Nidhi</a> is a Senior Partner Solutions Architect at AWS, specializing in data and analytics. She helps customers and partners build and optimize Snowflake workloads on AWS. Nidhi has extensive experience leading production releases and deployments, with focus on Data, AI, ML, generative AI, and Advanced Analytics.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Andries Engelbrecht" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/09/05/2024-profile-pic-100x97.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Andries Engelbrecht</h3> 
  <p><a href="https://www.linkedin.com/in/andries-engelbrecht-427b8b1/" rel="noopener" target="_blank">Andries</a> is a Principal Partner Solutions Engineer at Snowflake working with AWS. He supports product and service integrations, as well the development of joint solutions with AWS. Andries has over 25 years of experience in the field of data and analytics.</p>
 </div> 
</footer>

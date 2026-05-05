---
title: "Access Databricks Unity Catalog data using catalog federation in the AWS Glue Data Catalog"
url: "https://aws.amazon.com/blogs/big-data/access-databricks-unity-catalog-data-using-catalog-federation-in-the-aws-glue-data-catalog/"
date: "Mon, 12 Jan 2026 20:37:44 +0000"
author: "Srividya Parthasarathy"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p>AWS has launched the <a href="https://aws.amazon.com/blogs/big-data/introducing-catalog-federation-for-apache-iceberg-tables-in-the-aws-glue-data-catalog/" rel="noopener noreferrer" target="_blank">catalog federation</a> capability, enabling direct access to Apache Iceberg tables managed in Databricks Unity Catalog through the <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a>. With this integration, you can discover and query Unity Catalog data in Iceberg format using an Iceberg REST API endpoint, while maintaining granular access controls through <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a>. This approach significantly reduces operational overhead for managing catalog synchronization and associated costs by alleviating the need to replicate or duplicate datasets between platforms.</p> 
<p>In this post, we demonstrate how to set up catalog federation between the Glue Data Catalog and Databricks Unity Catalog, enabling data querying using AWS analytics services.</p> 
<h2>Use cases and key benefits</h2> 
<p>This federation capability is particularly valuable if you run multiple data platforms, because you can maintain your existing Iceberg catalog investments while using AWS analytics services. Catalog federation supports read operations and provides the following benefits:</p> 
<ul> 
 <li><strong>Interoperability</strong> – You can enable interoperability across different data platforms and tools through Iceberg REST APIs while preserving the value of your established technology investments.</li> 
 <li><strong>Cross-platform analytics</strong> – You can connect AWS analytics tools (<a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a>, <a href="https://aws.amazon.com/redshift/" rel="noopener noreferrer" target="_blank">Amazon Redshift</a>, Apache Spark) to query Iceberg and UniForm tables stored in Databricks Unity Catalog. It supports Databricks on AWS integration with the AWS Glue Iceberg REST Catalog for metadata retrieval, while using Lake Formation for permission management.</li> 
 <li><strong>Metadata management</strong> – The solution avoids manual catalog synchronization by making Databricks Unity Catalog databases and tables discoverable within the Data Catalog. You can implement unified governance through Lake Formation for fine-grained access control across federated catalog resources.</li> 
</ul> 
<h2>Solution overview</h2> 
<p>The solution uses catalog federation in the Data Catalog to integrate with Databricks Unity Catalog. The federated catalog created in AWS Glue mirrors the catalog objects in Databricks Unity Catalog and supports OAuth-based authentication. The solution is represented in the following diagram.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/06/BDB-5486-1.png" /></p> 
<p>The integration involves three high-level steps:</p> 
<ol> 
 <li>Set up an integration principal in Databricks Unity Catalog and provide required read access on catalog resources to this principal. Enable OAuth-based authentication for the integration principal.</li> 
 <li>Set up catalog federation to Databricks Unity Catalog in the Glue Data Catalog: 
  <ol type="a"> 
   <li>Create a federated catalog in the Data Catalog using an AWS Glue connection.</li> 
   <li>Create an AWS Glue connection that uses the credentials of the integration principal (in Step 1) to connect to Databricks Unity Catalog. Configure an <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) role with permission to <a href="https://aws.amazon.com/s3" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) locations where the Iceberg table data resides. In a cross-account scenario, make sure the bucket policy grants required access to this IAM role. </li> 
  </ol> </li> 
 <li>Discover Iceberg tables in federated catalogs using Lake Formation or AWS Glue APIs. During query operations, Lake Formation manages fine-grained permissions on federated resources and credential vending for access to the underlying data.</li> 
</ol> 
<p>In the following sections, we walk through the steps to integrate the Glue Data Catalog with Databricks Unity Catalog on AWS.</p> 
<h2>Prerequisites</h2> 
<p>To follow along with the solution presented in this post, you must have the following prerequisites:</p> 
<ul> 
 <li>Databricks Workspace (on AWS) with Databricks Unity Catalog configured.</li> 
 <li>An IAM role that is a Lake Formation data lake administrator in your AWS account. A data lake administrator is an IAM principal that can register S3 locations, access the Data Catalog, grant Lake Formation permissions to other users, and view <a href="https://aws.amazon.com/cloudtrail" rel="noopener noreferrer" target="_blank">AWS CloudTrail</a> logs. See <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/initial-LF-setup.html#create-data-lake-admin" rel="noopener noreferrer" target="_blank">Create a data lake administrator</a> for more information.</li> 
</ul> 
<h2>Configure Databricks Unity Catalog for external access</h2> 
<p>Catalog federation to a Databricks Unity Catalog uses the OAuth2 credentials of a Databricks service principal configured in the workspace admin settings. This authentication mechanism allows the Data Catalog to access the metadata of various objects (such as catalogs, databases, and tables) within Databricks Unity Catalog, based on the privileges associated with the service principal. For proper functionality, grant the service principal with the necessary permissions (read permission on catalog, schema, and tables) to read the metadata of these objects and allow access from external engines.</p> 
<p>Next, catalog federation enables discovery and query of Iceberg tables in your Databricks Unity Catalog. For reading delta tables, enable UniForm on a Delta Lake table in Databricks to generate Iceberg metadata. For more information, refer to <a href="https://docs.databricks.com/gcp/en/delta/uniform" rel="noopener noreferrer" target="_blank">Read Delta tables with Iceberg clients</a>.</p> 
<p>Follow the Databricks tutorial and <a href="https://docs.databricks.com/aws/en/admin/users-groups/manage-service-principals" rel="noopener noreferrer" target="_blank">documentation</a> to create the service principal and associated privileges in your Databricks workspace. For this post, we use a service principal named <code>integrationprincipal</code> that is configured with required permissions (SELECT, USE CATALOG, USE SCHEMA) on Databricks Unity Catalog objects and will be used for authentication to catalog instance.</p> 
<p>Catalog federation supports OAuth2 authentication, so enable OAuth for the service principal and note down the <code>client_id</code> and <code>client_secret</code> for later use.</p> 
<h2>Set up Data Catalog federation with Databricks Unity Catalog</h2> 
<p>Now that you have service principal access for Databricks Unity Catalog, you can set up catalog federation in the Data Catalog. To do so, you create an <a href="https://aws.amazon.com/secrets-manager/" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> secret and create an IAM role for catalog federation.</p> 
<h3>Create secret</h3> 
<p>Complete the following steps to create a secret:</p> 
<ol> 
 <li>Sign in to the <a href="http://aws.amazon.com/console" rel="noopener noreferrer" target="_blank">AWS Management Console</a> using an IAM role with access to Secrets Manager.</li> 
 <li>On the Secrets Manager console, choose <strong>Store a new secret</strong> and <strong>Other type of secret</strong>.</li> 
 <li>Set the key-value pair: 
  <ol type="a"> 
   <li>Key: <code>USER_MANAGED_CLIENT_APPLICATION_CLIENT_SECRET</code></li> 
   <li>Value: The client secret noted earlier</li> 
  </ol> </li> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>Enter a name for your secret (for this post, we use <code>dbx</code>).</li> 
 <li>Choose <strong>Store</strong>.</li> 
</ol> 
<h3>Create IAM role for catalog federation</h3> 
<p>As the catalog owner of a federated catalog in the Data Catalog, you can use Lake Formation to implement comprehensive access controls, including table filters, column filters, and row filters, as well as tag-based access for your data teams.</p> 
<p>Lake Formation requires an IAM role with permissions to access the underlying S3 locations of your external catalog.</p> 
<p>In this step, you create an IAM role that enables the AWS Glue connection to access Secrets Manager, optional virtual private cloud (VPC) configurations, and Lake Formation to manage credential vending for the S3 bucket and prefix:</p> 
<ul> 
 <li><strong>Secrets Manager access</strong> – The AWS Glue connection requires permissions to retrieve secret values from Secrets Manager for OAuth tokens stored for your Databricks Unity service connection.</li> 
 <li><strong>VPC access (optional)</strong> – When using VPC endpoints to restrict connectivity to your Databricks Unity account, the AWS Glue connection needs permissions to describe and utilize VPC network interfaces. This configuration provides secure, controlled access to both your stored credentials and network resources while maintaining proper isolation through VPC endpoints.</li> 
 <li><strong>S3 bucket and AWS KMS key permission</strong> – The AWS Glue connection requires Amazon S3 permissions to read certificates if used in the connection setup. Additionally, Lake Formation requires read permissions on the bucket and prefix where the remote catalog table data resides. If the data is encrypted using an <a href="http://aws.amazon.com/kms" rel="noopener noreferrer" target="_blank">AWS Key Management Service</a> (AWS KMS) key, additional AWS KMS permissions are required. </li> 
</ul> 
<p>Complete the following steps:</p> 
<ol> 
 <li>Create an <a href="https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html" rel="noopener noreferrer" target="_blank">IAM role</a> called <code>LFDataAccessRole</code> with the following policies: 
  <div class="hide-language"> 
   <pre><code class="lang-code">{
 "Version": "2012-10-17",
 &nbsp;&nbsp; &nbsp;"Statement": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"secretsmanager:GetSecretValue",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"secretsmanager:DescribeSecret"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"&lt;secrets manager ARN&gt;"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ec2:CreateNetworkInterface",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ec2:DeleteNetworkInterface",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ec2:DescribeNetworkInterfaces"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ArnEquals": {
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ec2:Vpc": "arn:aws:ec2:region:account-id:vpc/&lt;vpc-id&gt;", 
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ec2:Subnet": [ 
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:ec2:region:account-id:subnet/&lt;subnet-id&gt;" 
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; # Required when using custom cert to sign requests.
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"s3:GetObject"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:s3
:::&lt;bucketname&gt;/&lt;certpath&gt;"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{ # Required when using customer managed encryption key for s3 
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"kms:decrypt",
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"kms:encrypt"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"&lt;kmsKey&gt;"
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
 &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
 &nbsp;&nbsp; &nbsp;]
 }</code></pre> 
  </div> </li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/reference/iam/update-assume-role-policy.html" rel="noopener noreferrer" target="_blank">Configure the role</a> with the following trust policy: 
  <div class="hide-language"> 
   <pre><code class="lang-code">{
  &nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
  &nbsp;&nbsp; &nbsp;"Statement": [
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect":  "Allow",
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  &nbsp;"Service":&nbsp;["glue.amazonaws.com","lakeformation.amazonaws.com"]
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action":  "sts:AssumeRole"
  &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
  &nbsp;&nbsp; &nbsp;]
  }</code></pre> 
  </div> </li> 
</ol> 
<h2>Create federated catalog in Data Catalog</h2> 
<p>AWS Glue supports the <code>DATABRICKSICEBERGRESTCATALOG</code> connection type for connecting the Data Catalog with managed Databricks Unity Catalog. This AWS Glue connector supports OAuth2 authentication for discovering metadata in Databricks Unity Catalog.</p> 
<p>Complete the following steps to create the federated catalog:</p> 
<ol> 
 <li>Sign in to the console as a data lake admin.</li> 
 <li>On the Lake Formation console, choose <strong>Catalogs</strong> in the navigation pane.</li> 
 <li>Choose <strong>Create catalog</strong>.</li> 
 <li>For <strong>Name</strong>, enter a name for your catalog.</li> 
 <li>For <strong>Catalog name in Databricks</strong>, enter the name of a catalog existing in Databricks Unity Catalog.</li> 
 <li>For <strong>Connection name</strong>, enter a name for the AWS Glue connection.</li> 
 <li>For <strong>Workspace URL</strong>, enter the Unity Iceberg REST API URL (in format <code>https://</code>&lt;workspace-url&gt;<code>/cloud.databricks.com</code>).</li> 
 <li>For <strong>Authentication</strong>, provide the following information: 
  <ol type="a"> 
   <li>For <strong>Authentication type</strong>,<strong> </strong>choose <strong>OAuth2</strong>. Alternatively, you can choose <strong>Custom authentication</strong>. For <strong>Custom authentication</strong>, an access token is created, refreshed, and managed by the customer’s application or system and stored using Secrets Manager.</li> 
   <li>For <strong>Token URL</strong>, enter the token authentication server URL.</li> 
   <li>For <strong>OAuth Client ID</strong>, enter the <code>client_id</code> for <code>integrationprincipal</code>.</li> 
   <li>For <strong>OAuth Secret</strong>, enter the secret ARN that you created in the previous step. Alternatively, you can provide the <code>client_secret</code> directly.</li> 
   <li>For <strong>Token URL parameter map scope</strong>, provide the API scope supported.</li> 
  </ol> </li> 
 <li>If you have <a href="https://aws.amazon.com/privatelink/" rel="noopener noreferrer" target="_blank">AWS PrivateLink</a> set up or a proxy set up, you can provide network details under <strong>Settings for network configurations</strong>.</li> 
 <li>For <strong>Register Glue connection with Lake Formation</strong>, choose the IAM role (<code>LFDataAccessRole</code>) created earlier to manage data access using Lake Formation.</li> 
</ol> 
<p>When the setup is done using <a href="http://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI) commands, you have options to create two separate IAM roles:</p> 
<ul> 
 <li>IAM role with policies to access network and secrets, which AWS Glue assumes to manage authentication</li> 
 <li>IAM role with access to the S3 bucket, which Lake Formation assumes to manage credential vending for data access</li> 
</ul> 
<p>On the console, this setup is simplified with a single role having combined policies. For more details, refer to <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/catalog-federation-databricks.html" rel="noopener noreferrer" target="_blank">Federate to Databricks Unity Catalog</a>.</p> 
<ol start="11"> 
 <li>To test the connection, choose <strong>Run test</strong>.</li> 
 <li>You can proceed to create the catalog.</li> 
</ol> 
<p>After you create the catalog, you can see the databases and tables in Databricks Unity Catalog listed under the federated catalog. You can implement fine-grained access control on the tables by applying row and column filters using Lake Formation. The following video shows the catalog federation setup with Databricks Unity Catalog.</p> 
<div class="wp-video" style="width: 640px;">
 <video class="wp-video-shortcode" controls="controls" height="360" id="video-86726-2" preload="metadata" width="640">
  <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5486/dbxblog1.mp4?_=2" type="video/mp4" />
 </video>
</div> 
<h2>Discover and query the data using Athena</h2> 
<p>In this post, we show how to use the Athena query editor to discover and query the Databricks Unity Catalog tables. On the Athena console, run the following query to access the federated table:<code>SELECT * FROM "customerschema"."person" limit 10;</code>The following video demonstrates querying the federated table from Athena.</p> 
<div class="wp-video" style="width: 640px;">
 <video class="wp-video-shortcode" controls="controls" height="360" id="video-86726-3" preload="metadata" width="640">
  <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5486/dbxblogpart2.mp4?_=3" type="video/mp4" />
 </video>
</div> 
<p>If you use the Amazon Redshift query engine, you must create a resource link on the federated database and grant permission on the resource link to the user or role. This database resource link is automounted under <code>awsdatacatalog</code> based on the permission granted for the user or role and available for querying. For instructions, refer to <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/creating-resource-links.html" rel="noopener noreferrer" target="_blank">Creating resource links.</a></p> 
<h2>Clean up</h2> 
<p>To clean up your resources, complete the following steps:</p> 
<ol> 
 <li>Delete the catalog and namespace in Databricks Unity Catalog for this post.</li> 
 <li>Drop the resources in the Data Catalog and Lake Formation created for this post.</li> 
 <li>Delete the IAM roles and S3 buckets used for this post.</li> 
 <li>Delete any VPC and KMS keys if used for this post.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>In this post, we explored the key elements of catalog federation and its architectural design, illustrating the interaction between the AWS Glue Data Catalog and Databricks Unity Catalog through centralized authorization and credential distribution for protected data access. By removing the requirement for complicated synchronization workflows, catalog federation makes it possible to query Iceberg data on Amazon S3 directly at its source using AWS analytics services with data governance across multi-catalog platforms. Try out the solution for your own use case, and share your feedback and questions in the comments.</p> 
<hr /> 
<h3>About the Authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Srividya Parthasarathy" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/06/BDB-5486-4.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Srividya Parthasarathy</h3> 
  <p><a href="https://www.linkedin.com/in/srividya-parthasarathy-8b71bb32/" rel="noopener" target="_blank">Srividya</a> is a Senior Big Data Architect on the AWS Lake Formation team. She works with the product team and customers to build robust features and solutions for their analytical data platform. She enjoys building data mesh solutions and sharing them with the community.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Venkatavaradhan (Venkat) Viswanathan" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/06/BDB-5486-5.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Venkatavaradhan (Venkat) Viswanathan</h3> 
  <p><a href="https://www.linkedin.com/in/venkatavaradhanviswanathan/" rel="noopener" target="_blank">Venkat”</a> is a Global Partner Solutions Architect at Amazon Web Services. Venkat is a Technology Strategy Leader in Data, AI, ML, Generative AI, and Advanced Analytics. Venkat is a Global SME for Databricks and helps AWS customers design, build, secure, and optimize Databricks workloads on AWS.</p>
 </div> 
</footer>

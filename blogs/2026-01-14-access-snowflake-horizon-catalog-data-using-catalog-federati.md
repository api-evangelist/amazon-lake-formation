---
title: "Access Snowflake Horizon Catalog data using catalog federation in the AWS Glue Data Catalog"
url: "https://aws.amazon.com/blogs/big-data/access-snowflake-horizon-catalog-data-using-catalog-federation-in-the-aws-glue-data-catalog/"
date: "Wed, 14 Jan 2026 21:00:26 +0000"
author: "Andries Engelbrecht"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p><em>This is a guest post by Andries Engelbrecht, Principal Partner Solutions Engineer at Snowflake, in partnership with AWS.</em></p> 
<p>AWS announced a new catalog federation feature that allows you to directly access data from <a href="https://www.snowflake.com/en/product/features/horizon/" rel="noopener noreferrer" target="_blank">Snowflake Horizon Catalog</a> through the <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a>. This integration enables you to discover and query Horizon Catalog data in Iceberg format through REST endpoints while applying fine-grained access controls using <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a>. The new catalog federation combined with <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database" rel="noopener noreferrer" target="_blank">Snowflake’s catalog-linked database</a> feature means users can access data stored across AWS and Snowflake from a single point of entry, reducing data movement and associated costs by eliminating the need to duplicate data across platforms.</p> 
<p>In this post, we show you how to connect the AWS Glue Data Catalog to Snowflake Horizon Catalog and query the data using AWS analytics services. We cover how to set up catalogs in Horizon Catalog and configure required permissions, create and configure the federation connection in <a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a>, implement fine-grained access controls using AWS Lake Formation, and finally, query federated tables using <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a>. This step-by-step approach guides you through the complete process of establishing a integration between your Snowflake and AWS data environments.</p> 
<h2>Business examples and key benefits</h2> 
<p>Catalog federation enables several critical business scenarios while delivering key operational and strategic benefits.</p> 
<h3>Common examples</h3> 
<p>This federation capability addresses several key business scenarios:</p> 
<ul> 
 <li><strong>Governed, cross-platform analytics:</strong> Query data across AWS and Snowflake environments to improve data-driven decision making without data movement or duplication</li> 
 <li><strong>Data mesh implementation:</strong> Enable secure and federated data discovery while maintaining domain-oriented ownership</li> 
 <li><strong>Compliance management:</strong> Implement consistent access controls and auditing across platforms</li> 
</ul> 
<h3>Key benefits</h3> 
<ul> 
 <li><strong>Operational efficiency:</strong> Eliminate data duplication and reduce Extract Transform Load (ETL) workloads</li> 
 <li><strong>Enhanced security:</strong> Centralize access control through AWS Lake Formation with fine-grained permissions</li> 
 <li><strong>Cost optimization:</strong> Minimize data transfer and storage costs across platforms</li> 
 <li><strong>Improved agility:</strong> Enable faster time to insights with direct query access</li> 
 <li><strong>Simplified governance:</strong> Maintain unified compliance and audit framework</li> 
</ul> 
<h2>Solution overview</h2> 
<p>The solution uses catalog federation in the AWS Glue Data Catalog to integrate with Snowflake Horizon Catalog. This integration supports both Snowflake Horizon, where the catalog is internal to Snowflake, and external catalogs such as <a href="https://polaris.apache.org/" rel="noopener noreferrer" target="_blank">Apache Polaris</a>, <a href="https://docs.snowflake.com/en/user-guide/opencatalog/overview" rel="noopener noreferrer" target="_blank">Snowflake Open Catalog</a> (a managed service that hosts Apache Polaris), and others.</p> 
<p>The following diagram illustrates how AWS Glue Data Catalog federates with Snowflake Horizon Catalog, enabling customers to directly access Iceberg-format data managed by Snowflake Horizon Catalog through the Glue Data Catalog.</p> 
<p><img alt="Architecture diagram showing integration between AWS services and Snowflake using federated catalog connections through Apache Iceberg REST API." class="alignnone wp-image-86550 size-full" height="463" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-1.png" width="1042" /></p> 
<p>The integration works through three main components:</p> 
<ol> 
 <li><strong>Authentication</strong>: Uses OAuth2 credentials of Snowflake principal</li> 
 <li><strong>Access Control</strong>: <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a> manages fine-grained permissions</li> 
 <li><strong>Query Access</strong>: <a href="https://aws.amazon.com/big-data/datalakes-and-analytics/" rel="noopener noreferrer" target="_blank">AWS Analytics services</a> like <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a> can directly query the federated tables</li> 
</ol> 
<p>Now, we walk through the step-by-step process of setting up this integration.</p> 
<h2>Prerequisites</h2> 
<p>Before you begin, confirm you have the following:</p> 
<ul> 
 <li>A <a href="https://signup.snowflake.com/" rel="noopener noreferrer" target="_blank">Snowflake account</a>.</li> 
 <li>(Optional) A <a href="https://docs.snowflake.com/en/user-guide/opencatalog/create-open-catalog-account" rel="noopener noreferrer" target="_blank">Snowflake Open Catalog account</a>.</li> 
 <li>An <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> role that is a Lake Formation data lake administrator in your AWS account. A data lake administrator is an IAM principal that can register <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon S3</a> locations, access the Data Catalog, grant Lake Formation permissions to other users, and view <a href="https://aws.amazon.com/cloudtrail" rel="noopener noreferrer" target="_blank">AWS CloudTrail</a>. See <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/initial-LF-setup.html#create-data-lake-admin" rel="noopener noreferrer" target="_blank">Create a data lake administrator</a> for more information. This IAM role needs access to: 
  <ul> 
   <li><a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS IAM</a></li> 
   <li><a href="https://aws.amazon.com/secrets-manager/" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a></li> 
   <li><a href="https://aws.amazon.com/vpc/" rel="noopener noreferrer" target="_blank">Amazon VPC</a></li> 
   <li><a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a></li> 
   <li><a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a></li> 
   <li><a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a></li> 
   <li><a href="https://aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank">AWS KMS</a></li> 
  </ul> </li> 
 <li>Install or update the latest&nbsp;<a href="http://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface&nbsp;(CLI)</a> version to run the AWS CLI commands. For instructions, refer to&nbsp;<a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html" rel="noopener noreferrer" target="_blank">Installing or updating the latest version of the AWS CLI</a>.</li> 
</ul> 
<h2>Configure Snowflake Horizon Catalog for Iceberg external access</h2> 
<p>Snowflake Horizon Catalog already supports managing Iceberg tables. For this walkthrough, you need to create Snowflake-managed Iceberg tables with data stored in Amazon S3.</p> 
<p>Follow these steps in order:</p> 
<ol> 
 <li><strong>Create an external volume for S3:</strong> First, create an external volume that points to your S3 bucket where Iceberg table data is stored. Follow the instructions in <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-external-volume-s3" rel="noopener noreferrer" target="_blank">Create External Volume(s) for the Iceberg Tables on S3</a>.</li> 
 <li><strong>Create a database:</strong> Create a database to organize your tables. Refer to the <a href="https://docs.snowflake.com/en/sql-reference/sql/create-database" rel="noopener noreferrer" target="_blank">Snowflake database creation documentation</a>.</li> 
 <li><strong>Create a schema:</strong> Create a schema within your database following the <a href="https://docs.snowflake.com/en/sql-reference/sql/create-schema" rel="noopener noreferrer" target="_blank">Snowflake schema creation guide</a>.</li> 
 <li><strong>Create an Iceberg table:</strong> Create your Iceberg table using the external volume. Follow the instructions to <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-create#snowflake-managed" rel="noopener noreferrer" target="_blank">Create Iceberg Table</a>.</li> 
</ol> 
<p>After completing these steps, your Snowflake-managed Iceberg tables are ready to federate with AWS Glue Data Catalog.</p> 
<h2>Configure access control and authentication</h2> 
<p>To enable AWS Glue to access your Snowflake-managed Iceberg tables, you need to configure access control and obtain authentication credentials.</p> 
<h3>Step 1: Configure access control</h3> 
<p>Create a dedicated Snowflake role for external engine access to establish clear governance boundaries. Follow the instructions in <a href="https://docs.snowflake.com/user-guide/tables-iceberg-query-using-external-query-engine-snowflake-horizon#step-2-configure-access-control" rel="noopener noreferrer" target="_blank">Configure Access Control for external engines</a> and set up the appropriate permissions for your Iceberg tables.</p> 
<h3>Step 2: Obtain an access token</h3> 
<p><a href="https://docs.snowflake.com/user-guide/tables-iceberg-query-using-external-query-engine-snowflake-horizon#step-3-obtain-an-access-token-for-authentication" rel="noopener noreferrer" target="_blank">Generate an access token for authenticating AWS Glue to Snowflake Horizon Catalog</a>. Snowflake supports three authentication mechanisms:</p> 
<ul> 
 <li>External OAuth</li> 
 <li>Key-pair authentication</li> 
 <li>Programmatic Access Token (PAT)</li> 
</ul> 
<p>Choose the authentication method that best fits your security requirements and follow the corresponding Snowflake documentation to generate your credentials.</p> 
<p>Catalog Federation supports OAuth or custom authentication. For details on using OAuth refer to <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/catalog-federation-snowflake.html" rel="noopener noreferrer" target="_blank">Federate to Snowflake Iceberg Catalog</a>.</p> 
<p>For this post, we use custom authentication and generate access token using PAT. Replace <code>role_name</code> with the principal role and <code>token_value</code> with the principal’s Programmatic Access Token.</p> 
<div class="hide-language"> 
 <pre><code class="lang-shell">curl --location 'https://&lt;accountidentifier&gt;.snowflakecomputing.com/polaris/api/catalog/v1/oauth/tokens' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=client_credentials' \
--data-urlencode 'scope=session:role:&lt;role_name&gt;' \
--data-urlencode 'client_secret=&lt;token_value&gt;'</code></pre> 
</div> 
<p>Note down the access token that is generated.</p> 
<h3>Step 3: Enable catalog federation</h3> 
<p>With access control configured and authentication credentials in hand, AWS Glue Catalog Federation can now connect to and <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-query-using-external-query-engine-snowflake-horizon" rel="noopener noreferrer" target="_blank">access Snowflake’s Horizon Catalog</a>.</p> 
<h3>Optional: Snowflake Open Catalog configuration</h3> 
<p>If you prefer to use Snowflake Open Catalog for Iceberg external access instead, refer to <a href="https://docs.snowflake.com/en/user-guide/tables-iceberg-open-catalog-sync" rel="noopener noreferrer" target="_blank">Sync a Snowflake-managed table with Snowflake Open Catalog</a> for alternative setup instructions.</p> 
<h2>Setup Glue Catalog federation with Snowflake Horizon Catalog</h2> 
<h4>Create a secret on AWS Secrets Manager</h4> 
<p>Log in to AWS console using the IAM role that has access to AWS Secrets Manager. Open Secrets Manager:</p> 
<ul> 
 <li>Choose <strong>Store a new secret</strong> and select <strong>Other type of secret</strong> for the secret type.</li> 
 <li>Set the key-value pair: 
  <ul> 
   <li>Key: <code>BEARER_TOKEN</code></li> 
   <li>Value: The access token noted earlier</li> 
  </ul> </li> 
 <li>Choose <strong>Next</strong> and provide the secret name as horizon-secret.</li> 
 <li>Complete the setup by choosing <strong>Store</strong>.</li> 
</ul> 
<p>Alternatively, you can use the CLI to create the secret by running the following command.</p> 
<p>Replace <code>your-access-token</code> and <code>your-region</code> with your actual values:</p> 
<div class="hide-language"> 
 <pre><code class="lang-shell">aws secretsmanager create-secret \
    --name horizon-secret \
    --description "Snowflake Horizon access token" \
    --secret-string '{
        "BEARER_TOKEN": "your-access-token"
    }' \
    --region your-region</code></pre> 
</div> 
<h3>Create IAM role for catalog federation</h3> 
<p>As the catalog owner of a federated catalog in AWS Glue Data Catalog, you can use Lake Formation to implement comprehensive access controls for your data teams:</p> 
<h4><strong>Access control options</strong></h4> 
<p>You can implement access controls at different granularity levels depending on your governance needs:</p> 
<ul> 
 <li>Coarse-grained: Table-level permissions</li> 
 <li>Fine-grained: Column-level, row-level, and cell-level filtering</li> 
 <li>Tag-based: Dynamic access based on data classification tags</li> 
</ul> 
<p>Lake Formation requires an IAM role with permissions to access the underlying S3 locations of your external catalog.</p> 
<p>Create an IAM role that enables the Glue Connection to access AWS Secrets Manager, VPC configurations (optional) and Lake formation to manage credential vending for S3 bucket/prefix.</p> 
<p>Required permissions</p> 
<ol> 
 <li><strong>Secrets Manager access:</strong> The Glue connection requires permissions to retrieve secret values from Secrets Manager for OAuth tokens stored for your Snowflake service connection.</li> 
 <li><a href="https://aws.amazon.com/vpc/" rel="noopener noreferrer" target="_blank"><strong>Amazon Virtual Private Cloud (VPC)</strong></a><strong> Access (optional):</strong> When using VPC endpoints to restrict connectivity to your Snowflake Open Catalog account, the Glue connection needs permissions to describe and use VPC network interfaces. This configuration ensures secure, controlled access to both your stored credentials and network resources while maintaining proper isolation through VPC endpoints.</li> 
 <li><strong>S3 bucket and </strong><a href="https://aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank"><strong>AWS Key Management Service (KMS)</strong></a><strong> key permission: </strong>The Glue connection requires S3 permissions to read certificates if used in the connection setup. Additionally, Lake Formation requires read permissions on the bucket/prefix where the remote catalog table data resides. If the data is encrypted using a KMS key, additional KMS permissions are required.</li> 
</ol> 
<p><strong>Setup steps:</strong></p> 
<p>Run the following command using AWS CLI by replacing the placeholder with your setup information:</p> 
<p>Create a JSON file (e.g.,&nbsp;<code>trust-policy.json</code>) with the following structure:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Service":&nbsp;["glue.amazonaws.com","lakeformation.amazonaws.com"]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "sts:AssumeRole"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<p>Use the&nbsp;<code>aws iam create-role</code>&nbsp;command, referencing the trust policy file:</p> 
<div class="hide-language"> 
 <pre><code class="lang-shell">aws iam create-role \
    --role-name LFDataAccessRole \
    --assume-role-policy-document file://&lt;path_file_downloaded&gt;/trust-policy.json </code></pre> 
</div> 
<p>First, create a JSON file (such as,&nbsp;<code>permissions-policy.json</code>) for the permissions:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">
{
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
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:s3:::&lt;bucketname&gt;/&lt;certpath&gt;"
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
</div> 
<p>Then, attach it to the role:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws iam put-role-policy \
--role-name LFDataAccessRole \
--policy-name myaccesspolicies \
--policy-document file://&lt;path_file_downloaded&gt;/permissions- policy.json</code></pre> 
</div> 
<h3>Create federated catalog in Glue Data Catalog</h3> 
<p>AWS Glue supports the <code>SNOWFLAKEICEBERGRESTCATALOG</code> connection type for connecting Glue Data Catalog with Snowflake Horizon Catalog and Snowflake Open Catalog. This Glue connector supports OAuth2 authentication and includes additional configuration parameters like <code>CASING_TYPE</code> to customize how AWS Glue Data Catalog discovers metadata in the Snowflake Horizon Catalog accounts.</p> 
<p>Log in to your AWS console as a data lake admin and open the AWS Lake Formation console.</p> 
<ol> 
 <li>Choose <strong>Catalog</strong> in the left navigation pane and select <strong>Create catalog</strong>.</li> 
 <li>Choose the data source as <strong>Snowflake Horizon Catalog.</strong><br /> <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-2.png"><img alt="AWS Lake Formation console screenshot showing Step 1 of catalog creation wizard with five federation type options, Snowflake Horizon Catalog selected." class="alignnone wp-image-86551 size-full" height="617" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-2.png" width="1430" /></a></li> 
 <li>Provide the following information: 
  <ul> 
   <li><strong>Name</strong>: Name of the federated catalog in Glue Catalog. For this post, we use federated_lakehousedb</li> 
   <li><strong>Catalog name in Snowflake</strong>: Catalog name existing in Snowflake Horizon Catalog, this should match exact name in Horizon catalog. For this post, we use LAKEHOUSEDB</li> 
   <li>For <strong>Connection details</strong>, choose <strong>New connection configurations</strong>: 
    <ul> 
     <li><strong>Connection name</strong>: Name for the glue connection. For this post, we use federatedconnection1.</li> 
     <li><strong>Workspace URL</strong>: Horizon IRC url (format: https://&lt;account_identifier&gt;.snowflakecomputing.com)</li> 
     <li><strong>Casing type</strong>: choose Uppercase only</li> 
     <li><strong>Authentication</strong>: 
      <ul> 
       <li><strong>Authentication type</strong>: choose <strong>Custom</strong>. Alternatively, you can select <strong>OAuth2 authentication</strong>. For Custom authentication, an access token is created, refreshed, and managed by the customer’s application or system and stored using AWS Secrets Manager.</li> 
       <li><strong>OAuth Secret</strong>: Provide the secret manager ARN that was created in the previous step.</li> 
      </ul> </li> 
    </ul> </li> 
  </ul> </li> 
</ol> 
<ul> 
 <li>If you have <a href="https://aws.amazon.com/privatelink/" rel="noopener noreferrer" target="_blank">AWS PrivateLink</a> setup and/or a proxy setup, you can provide network details under <strong>Settings for network configurations </strong>(optional).</li> 
 <li><strong>For Register Glue connection with Lake Formation:</strong> 
  <ul> 
   <li>Choose the IAM role created earlier(LFDataAccessRole) to manage data access using Lake Formation.</li> 
  </ul> </li> 
</ul> 
<p>To test the connection, choose <strong>Run test</strong>. After the connection information is validated, it shows as successful.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-3.png"><img alt="Green success banner displaying &quot;Connection test successful&quot; with checkmark icon, confirming valid AWS configuration." class="alignnone wp-image-86552 size-full" height="119" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-3.png" width="1432" /></a></p> 
<p>You can now create the catalog by selecting <strong>Create catalog</strong>.</p> 
<p>Alternatively, you can use AWS CLI to create connection and catalog using example commands:</p> 
<div class="hide-language"> 
 <pre><code class="lang-shell">aws glue create-connection \
--connection-input '{
"Name": "federatedconnection1",
"ConnectionType": "SNOWFLAKEICEBERGRESTCATALOG",
"ConnectionProperties": {
    "INSTANCE_URL": "&lt;your-snowflake-account-URL&gt;",
    "ROLE_ARN": "&lt; ARN_of_LFDataAccessRole&gt;",
    "CATALOG_CASING_FILTER": "UPPERCASE_ONLY"
},
"AuthenticationConfiguration": {
    "AuthenticationType": "CUSTOM",
    "SecretArn": "arn:aws:secretsmanager:&lt;your-aws-region&gt;:&lt;your-aws-account-id&gt;:secret:horizon-secret"
}
}' \
--region &lt;your-aws-region&gt;
aws lakeformation register-resource \
    --resource-arn &lt;ARN_of_federatedconnection1_connection&gt; \
    --role-arn &lt;ARN_of_LFDataAccessRole&gt; \
    --with-federation \
    --with-privileged-access \
    --region &lt;your-aws-region&gt;
aws glue create-catalog \
    --name federated_lakehousedb \
    --catalog-input '{
    "FederatedCatalog": {
        "Identifier": "LAKEHOUSEDB",
        "ConnectionName": “federatedconnection1 "
    },
    "CreateTableDefaultPermissions": [],
    "CreateDatabaseDefaultPermissions": []
}'</code></pre> 
</div> 
<div class="wp-video" style="width: 640px;">
 <video class="wp-video-shortcode" controls="controls" height="360" id="video-86547-1" preload="metadata" width="640">
  <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5498/BDB-5498-bloghorizondemo.mp4?_=1" type="video/mp4" />
 </video>
</div> 
<p>After the catalog is created, the Horizon databases and tables are listed under the federated catalog.</p> 
<p>You can implement fine grained access control on the tables by applying row/column filter using Lake Formation.</p> 
<h3>Query the data using Athena query editor:</h3> 
<p>Open the Amazon Athena console and run the following query to access the federated Horizon table:</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">SELECT * FROM "public"."customer" limit 10;</code></pre> 
</div> 
<h2>Clean up</h2> 
<p>To clean up your resources, complete the following steps:</p> 
<ol> 
 <li><a href="https://docs.snowflake.com/en/sql-reference/sql/drop-database" rel="noopener noreferrer" target="_blank">Drop the Snowflake Database with Cascade.</a></li> 
 <li><a href="https://docs.snowflake.com/en/sql-reference/sql/drop-external-volume" rel="noopener noreferrer" target="_blank">Drop External Volume</a> created for Iceberg Tables on S3.</li> 
 <li><a href="https://docs.aws.amazon.com/lake-formation/latest/dg/delete-glue-fed-catalog.html" rel="noopener noreferrer" target="_blank">Drop the resources in Glue Data Catalog and Lake Formation</a> created for this post.</li> 
 <li><a href="https://docs.aws.amazon.com/IAM/latest/APIReference/API_DeleteRole.html" rel="noopener noreferrer" target="_blank">Delete the IAM roles</a> and <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-bucket.html" rel="noopener noreferrer" target="_blank">S3 buckets</a> used for this post.</li> 
 <li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html" rel="noopener noreferrer" target="_blank">Delete any VPC</a>, <a href="https://docs.aws.amazon.com/kms/latest/developerguide/deleting-keys.html" rel="noopener noreferrer" target="_blank">KMS keys</a> if used for this post setup.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to establish a secure connection between AWS Analytics services and Snowflake Horizon Catalog, enabling you to access your data from a single connected and governed view. You learned how to:</p> 
<ul> 
 <li>Configure catalog federation between AWS Glue Data Catalog and Snowflake Horizon Catalog</li> 
 <li>Set up OAuth2 authentication for secure access</li> 
 <li>Grant access to Iceberg table in Snowflake Horizon Catalog using AWS Lake Formation</li> 
 <li>Query federated tables using Amazon Athena</li> 
</ul> 
<p>You can follow the same steps to establish a secure connection with open-source catalog options such as Snowflake Open Catalog, a managed service for Apache Iceberg. Remember to clean up any resources you created while following this tutorial to avoid ongoing charges.</p> 
<p>To further explore this solution in your environment, consider the following resources:</p> 
<ul> 
 <li><a href="https://aws.amazon.com/about-aws/whats-new/2025/11/aws-glue-catalog-federation-remote-apache-iceberg-catalogs/" rel="noopener noreferrer" target="_blank">AWS Glue announces catalog federation for remote Apache Iceberg catalogs</a></li> 
 <li><a href="https://docs.aws.amazon.com/lake-formation/latest/dg/catalog-federation-snowflake.html" rel="noopener noreferrer" target="_blank">Federate to Snowflake Iceberg Catalog</a></li> 
 <li><a href="https://aws.amazon.com/blogs/big-data/introducing-catalog-federation-for-apache-iceberg-tables-in-the-aws-glue-data-catalog/" rel="noopener noreferrer" target="_blank">Introducing catalog federation for Apache Iceberg tables in the AWS Glue Data Catalog</a></li> 
</ul> 
<p>These resources can help you to implement and optimize this integration pattern for your specific use case. As you begin this journey, remember to start small, validate your architecture with test data, and gradually scale your implementation based on your organization’s needs. Stay tuned for future workshops and resources.</p> 
<hr /> 
<h3>About the authors</h3> 
<p>&nbsp;</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Andries Engelbrecht" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-8.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Andries Engelbrecht</h3> 
  <p><a href="https://www.linkedin.com/in/andries-engelbrecht-427b8b1/" rel="noopener" target="_blank">Andries</a> is a Principal Partner Solutions Engineer at Snowflake working with AWS. He supports product and service integrations, as well the development of joint solutions with AWS. Andries has over 25 years of experience in the field of data and analytics.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Nidhi Gupta" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-5-1.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Nidhi Gupta</h3> 
  <p><a href="https://www.linkedin.com/in/nidhi-gupta-5b80874/" rel="noopener" target="_blank">Nidhi</a> is a Senior Partner Solutions Architect at AWS, specializing in data analytics and AI. She helps customers and partners build and optimize Snowflake workloads on AWS. Nidhi has extensive experience leading development, production releases and deployments, with focus on Data, AI, ML, generative AI, and Advanced Analytics.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Srividya Parthasarathy" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-6.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Srividya Parthasarathy</h3> 
  <p><a href="https://www.linkedin.com/in/srividya-parthasarathy-8b71bb32/" rel="noopener" target="_blank">Srividya</a> is a Senior Big Data Architect on the AWS Lake Formation team. She works with the product team and customers to build robust features and solutions for their analytical data platform. She enjoys building data mesh solutions and sharing them with the community.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Pratik Das" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/01/05/bdb-5498-7.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Pratik Das</h3> 
  <p><a href="https://www.linkedin.com/in/das-pratik/" rel="noopener" target="_blank">Pratik</a> is a Senior Product Manager with AWS Lake Formation. He is passionate about all things data and works with customers to understand their requirements and build delightful experiences. He has a background in building data-driven solutions and machine learning systems.</p>
 </div> 
</footer> 
<p>&nbsp;</p>

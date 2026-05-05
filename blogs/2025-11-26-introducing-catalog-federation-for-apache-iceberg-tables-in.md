---
title: "Introducing catalog federation for Apache Iceberg tables in the AWS Glue Data Catalog"
url: "https://aws.amazon.com/blogs/big-data/introducing-catalog-federation-for-apache-iceberg-tables-in-the-aws-glue-data-catalog/"
date: "Wed, 26 Nov 2025 22:08:07 +0000"
author: "Debika D"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p>Apache Iceberg has become the standard choice of open table format for organizations seeking robust and reliable analytics at scale. However, enterprises increasingly find themselves navigating complex multi-vendor landscapes with disparate catalog systems. Managing data across these has become a major challenge for organizations operating in multi-vendor environments. This fragmentation drives significant operational complexity, particularly around access control and governance. Customers using AWS analytics services such as <a href="https://aws.amazon.com/redshift" rel="noopener noreferrer" target="_blank">Amazon Redshift</a>, <a href="https://aws.amazon.com/emr/" rel="noopener noreferrer" target="_blank">Amazon EMR</a>, <a href="https://aws.amazon.com/athena" rel="noopener noreferrer" target="_blank">Amazon Athena</a>, <a href="https://aws.amazon.com/sagemaker/" rel="noopener noreferrer" target="_blank">Amazon SageMaker</a>, and <a href="https://aws.amazon.com/glue" rel="noopener noreferrer" target="_blank">AWS Glue</a> to analyze Iceberg tables in the <a href="https://docs.aws.amazon.com/prescriptive-guidance/latest/serverless-etl-aws-glue/aws-glue-data-catalog.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a> want to get the same price-performance for workloads in remote catalogs. Simply migrating or replacing these remote&nbsp;catalogs isn’t practical, leaving teams to implement and maintain synchronization processes that continuously replicate metadata across systems, creating operational overhead, escalating costs, and risking data inconsistencies.</p> 
<p>AWS Glue now supports catalog federation for remote Iceberg tables in the Data Catalog. With catalog federation, you can query remote Iceberg tables, stored in <a href="https://aws.amazon.com/s3" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3) and cataloged in remote Iceberg catalogs, using AWS analytics engines and without moving or duplicating tables. After a remote catalog is integrated, AWS Glue always fetch the latest metadata in the background, so you always have access to the Iceberg metadata through your preferred AWS analytics services. This capability supports both coarse-grained access control and fine-grained permissions through <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a>, giving you the flexibility on how and when remote Iceberg tables are shared with data consumers. With integration for Snowflake Polaris Catalog, Databricks Unity Catalog, and other custom catalogs supporting Iceberg REST specifications, you can federate to remote catalogs, discover databases and tables, configure access permissions, and begin querying remote Iceberg data.</p> 
<p>In this post, we discuss how to get started with catalog federation for Iceberg tables in the Data Catalog.</p> 
<h2>Solution overview</h2> 
<p>Catalog federation uses the Data Catalog to communicate with remote catalog systems to discover catalog objects and Lake Formation to authorize access to their data in Amazon S3. When you query a remote Iceberg table, the Data Catalog discovers the latest table information in the remote catalog at query runtime, getting the table’s S3 location, current schema, and partition information. Your analytics engine (Athena, EMR, or Redshift) then uses this information to access Iceberg data files directly from Amazon S3. And Lake Formation manages access to the table by vending scoped credentials to the table data stored in Amazon S3, allowing the engines to apply fine-grained permissions to the federated table. This approach avoids metadata and data duplication while providing real-time access to remote Iceberg tables through your preferred AWS analytics engines.</p> 
<p>The Data Catalog facilitates connectivity to remote&nbsp;catalog systems that support Apache Iceberg by establishing an AWS Glue connection with the remote catalog endpoint. You can connect the Data Catalog to remote Iceberg REST catalogs using OAuth2 or custom authentication mechanisms using an access token.&nbsp;During integration, administrators configure a principal (service account or identity) with the appropriate permissions to access resources in the remote catalog. The AWS Glue connection object uses this configured principal’s credentials to authenticate and access metadata in the remote catalog server. You can also connect the Data Catalog to remote catalogs that use a private link or proxy for isolating and restricting network access. After it’s connected, this integration uses the standardized Iceberg REST API specification to retrieve the most current table metadata information from these remote catalogs. AWS Glue onboards these remote catalogs as federated catalogs within its own catalog infrastructure, enabling unified metadata access across multiple catalog systems.</p> 
<p>Lake Formation serves as the centralized authorization layer for managing user access to federated catalog resources. When users attempt to access tables and databases in federated catalogs, Lake Formation evaluates their permissions and enforces fine-grained access control policies.</p> 
<p>Beyond metadata authorization, the catalog federation also manages secure access to the actual underlying data files. It accomplishes this through credential vending mechanisms that issue temporary, scope-limited credentials. AWS Glue federated catalogs work with your preferred AWS analytics engines and query services, enabling consistent metadata access and unified data governance across your analytics workloads.</p> 
<p>In the following sections, we walk through the steps to integrate the Data Catalog with your remote catalog server:</p> 
<ol> 
 <li>Set up an integration principal in the remote catalog and provide required access on catalog resources to this principal. Enable OAuth based authentication for the integration principal.</li> 
 <li>Create a federated catalog in the Data Catalog using the AWS Glue connection. Create an AWS Glue connection that uses the credentials of the integration principal (in Step1) to connect to the Iceberg REST endpoint of the remote catalog. Configure an <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) role with permission to S3 locations where the remote table data resides. In a cross-account scenario, make sure the bucket policy grants required access to this IAM role. This federated catalog mirrors the catalog object in your remote catalog server.</li> 
 <li>Discover Iceberg tables in federated catalogs using Lake Formation or AWS Glue APIs. Query Iceberg tables using AWS analytics engines. During query operations, Lake Formation manages fine-grained permission on federated resources and credential vending to underlying data for the end-users.</li> 
</ol> 
<h3>Prerequisites</h3> 
<p>Before you begin, verify you have the following setup in AWS:</p> 
<ul> 
 <li>An <a href="https://aws.amazon.com/account/" rel="noopener noreferrer" target="_blank">AWS account</a>.</li> 
 <li>The <a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI) version 2.31.38 or later installed and configured.</li> 
 <li>An IAM admin role or user with appropriate permissions to the following services: 
  <ul> 
   <li>IAM</li> 
   <li>AWS Glue Data Catalog</li> 
   <li>Amazon S3</li> 
   <li>AWS Lake Formation</li> 
   <li>AWS Secrets manager</li> 
   <li>Amazon Athena</li> 
  </ul> </li> 
 <li>Create a data lake admin. For instructions, see <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/initial-lf-config.html#create-data-lake-admin" rel="noopener noreferrer" target="_blank">Create a data lake administrator</a>.</li> 
</ul> 
<h4>Set up authentication credentials in remote Iceberg catalog</h4> 
<p>Catalog federation to a remote Iceberg catalog uses the OAuth2 credentials of the principal configured with metadata access. This authentication mechanism allows the AWS Glue Data Catalog to access the metadata of various objects (such as databases, and tables) within the remote&nbsp;catalogs, based on the privileges associated with the principal. To support proper functionality, you must grant the principal with the necessary permissions to read the metadata of these objects. Generate the <code>CLIENT_ID</code> and <code>CLIENT_SECRET</code> to enable OAuth based authentication for the integration principal.</p> 
<h4>Create AWS Glue catalog federation using connection to remote Iceberg catalog</h4> 
<p>Create a federated catalog in the Data Catalog that mirrors a catalog object in the remote Iceberg catalog server and is used by the AWS Glue service to federate metadata queries such as <code>ListDatabases</code>, <code>ListTables</code>, and <code>GetTable</code> to the remote catalog. As data lake administrator, you can create a federated catalog in the Data Catalog using an AWS Glue connection object that is registered with AWS Lake Formation.</p> 
<p><strong>Configure&nbsp;data source connection for AWS Glue connection</strong></p> 
<p>Catalog federation uses an AWS Glue connection for metadata access when you provide authentication and Iceberg REST API endpoint configurations in the remote catalog. The AWS Glue connection supports OAuth2 or custom as the authentication method.</p> 
<p><strong>Connect using OAuth2 authentication</strong></p> 
<p>For the OAuth2 authentication method, you can provide a client secret either directly as input or stored in <a href="https://aws.amazon.com/secrets-manager/" rel="noopener" target="_blank">AWS Secrets Manager</a> and used by the AWS Glue connection object during authentication. AWS Glue internally manages the token refresh upon expiration. To store the client secret in Secrets manager, complete the following steps:</p> 
<ol> 
 <li>On the Secrets Manager console, choose <strong>Secrets</strong> in the navigation pane.</li> 
 <li>Choose <strong>Store a new secret</strong>.</li> 
 <li>Choose <strong>Other type of secret</strong>, provide the key name as <code>USER_MANAGED_CLIENT_APPLICATION_CLIENT_SECRET</code>, and enter the client secret value.</li> 
 <li>Choose <strong>Next </strong>and provide a name for the secret.</li> 
 <li>Choose <strong>Next</strong> and choose <strong>Store</strong> to save the secret.</li> 
</ol> 
<p><strong>Connect using custom&nbsp;authentication</strong></p> 
<p>For custom authentication, use Secrets Manager to store and retrieve the access token. This access token is created, refreshed, and managed by the customer’s application or system, providing proper control and management over the authentication process. To store the access token in Secrets Manager, complete the following steps:</p> 
<ol> 
 <li>On the Secrets Manager console, choose <strong>Secrets</strong> in the navigation pane.</li> 
 <li>Choose <strong>Store a new secret</strong>.</li> 
 <li>Choose <strong>Other type of secret</strong> and provide the key name as <code>BEARER_TOKEN</code> with the value noted as the access token of the integration principal.</li> 
 <li>Choose <strong>Next </strong>and provide a name for the secret.</li> 
 <li>Choose <strong>Next</strong> and choose <strong>Store</strong> to save the secret.</li> 
</ol> 
<p><strong>Register AWS Glue connection with Lake Formation</strong></p> 
<p>Create an IAM role that Lake Formation can use to vend credentials and attach permission on S3 bucket prefixes where the Iceberg tables are stored. Optionally, if you’re using Secrets Manager to store the client secret or are using a network configuration, you can add permissions for those services to this role. For instruction, refer to <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/catalog-federation.html" rel="noopener noreferrer" target="_blank">Catalog federation to remote Iceberg catalogs</a>.</p> 
<p>Complete the following steps to register the connection:</p> 
<ol> 
 <li>On the Lake Formation console, choose <strong>Catalogs</strong> in the navigation pane.</li> 
 <li>Choose <strong>Create catalog</strong> and select the data source.</li> 
 <li>Provide the federated catalog details: 
  <ol type="a"> 
   <li>Name of the federated catalog.</li> 
   <li>Catalog name in the remote catalog server and this needs to match the exact catalog name in remote catalog.</li> 
  </ol> </li> 
 <li>Provide AWS Glue connection details. To reuse an existing connection, choose <strong>Select existing connection</strong> and choose the connection to reuse. For a first-time setup, choose <strong>Input new connection configuration</strong> and provide the following information: 
  <ol type="a"> 
   <li>Provide the AWS Glue connection name.</li> 
   <li>Provide the remote catalog Iceberg REST API endpoint.</li> 
   <li>Specify the catalog object casing type. The connection can support uppercase objects through the object hierarchy or lowercase objects.</li> 
   <li>Configure authentication parameters: 
    <ol type="i"> 
     <li>For OAuth2: Provide the client ID and client secret directly or choose the secret where the client secret is stored, token authorization URL, and scope mapped to the credential.</li> 
     <li>For custom: Provide the secret managed by Secrets Manager where the access token is stored.</li> 
     <li>Network configuration: If you have a network and/or proxy setup, you can provide this information. Otherwise, leave this section as default.</li> 
    </ol> </li> 
  </ol> </li> 
 <li>Register the connection with Lake Formation using the IAM role with access to the bucket where the remote table metadata and data is stored.</li> 
 <li>Verify the connection by choosing <strong>Run test</strong>.</li> 
 <li>After the test is successful, create the catalog.<br /> 
  <div class="wp-video" style="width: 640px;">
   <video class="wp-video-shortcode" controls="controls" height="360" id="video-85674-4" preload="metadata" width="640">
    <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5682/catalogdemo.mp4?_=4" type="video/mp4" />
   </video>
  </div></li> 
</ol> 
<p>You can now discover remote objects under the federated catalog.&nbsp;You can onboard other remote catalogs by reusing the existing connection configured to the same external catalog instance.</p> 
<h4>Query the federated catalog objects using AWS analytical engines</h4> 
<p>As the data lake administrator, you can now manage access control on databases and tables in a federated catalog using AWS Lake Formation. You can also use tag-based access control to scale your permission model by tagging the resource based on the access control mechanism.</p> 
<div class="wp-video" style="width: 640px;">
 <video class="wp-video-shortcode" controls="controls" height="360" id="video-85674-5" preload="metadata" width="640">
  <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5682/allgrants.mp4?_=5" type="video/mp4" />
 </video>
</div> 
<p>After permissions are granted, an IAM principal or an IAM user can access the federated tables using AWS analytical services including Athena, Amazon Redshift, Amazon EMR, and Amazon SageMaker. Query the federated Iceberg table using Athena as shown in the following example.</p> 
<div class="wp-video" style="width: 640px;">
 <video class="wp-video-shortcode" controls="controls" height="360" id="video-85674-6" preload="metadata" width="640">
  <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5682/athena.mp4?_=6" type="video/mp4" />
 </video>
</div> 
<h2>Clean up</h2> 
<p>To avoid incurring ongoing charges, complete the following steps to clean up the resources created during this walkthrough:</p> 
<ol> 
 <li>Delete the federated catalog in the Data Catalog: 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws glue delete-catalog \
&nbsp;&nbsp; &nbsp;--name <span style="color: #ff0000;">&lt;your-federated-catalog-name&gt;</span></code></pre> 
  </div> </li> 
 <li>Deregister the AWS Glue connection from Lake Formation: 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws lakeformation deregister-resource \
&nbsp;&nbsp; &nbsp;--resource-arn <span style="color: #ff0000;">&lt;your-glue-connector-arn&gt;</span></code></pre> 
  </div> </li> 
 <li>Revoke Lake Formation permissions (if any were granted): 
  <div class="hide-language"> 
   <pre><code class="lang-code"># List existing permissions first
aws lakeformation list-permissions \
&nbsp;&nbsp; &nbsp;--catalog-id <span style="color: #ff0000;">&lt;your-account-id&gt;</span> \
&nbsp;&nbsp; &nbsp;--resource '{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Catalog": {}
&nbsp;&nbsp; &nbsp;}'

# Revoke permissions as needed
aws lakeformation revoke-permissions \
&nbsp;&nbsp; &nbsp;--principal '{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"DataLakePrincipalIdentifier": "<span style="color: #ff0000;">&lt;principal-arn&gt;</span>"
&nbsp;&nbsp; &nbsp;}' \
&nbsp;&nbsp; &nbsp;--resource '{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Database": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"CatalogId": "<span style="color: #ff0000;">&lt;catalog-id&gt;</span>",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Name": "&lt;database-name&gt;"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;}' \
&nbsp;&nbsp; &nbsp;--permissions ["SELECT", "DESCRIBE"]</code></pre> 
  </div> </li> 
 <li>Delete the AWS Glue connection: 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws glue delete-connection \
&nbsp;&nbsp; &nbsp;--connection-name <span style="color: #ff0000;">&lt;your-glue-connection-to-snowflake-account&gt;</span></code></pre> 
  </div> </li> 
 <li>Delete IAM roles and policies associated with Lake Formation and the AWS Glue connection: 
  <div class="hide-language"> 
   <pre><code class="lang-code"># Detach policies from the role
aws iam detach-role-policy \
&nbsp;&nbsp; &nbsp;--role-name <span style="color: #ff0000;">&lt;your-lakeformation-role-name&gt;</span> \
&nbsp;&nbsp; &nbsp;--policy-arn <span style="color: #ff0000;">&lt;your-lakeformation-policy-arn&gt;</span>

# Delete the custom policy
aws iam delete-policy \
&nbsp;&nbsp; &nbsp;--policy-arn <span style="color: #ff0000;">&lt;your-lakeformation-policy-arn&gt;</span>

# Delete the role
aws iam delete-role \
&nbsp;&nbsp; &nbsp;--role-name <span style="color: #ff0000;">&lt;your-lakeformation-role-name&gt;</span>
# Detach policies from the role
aws iam detach-role-policy \
&nbsp;&nbsp; &nbsp;--role-name <span style="color: #ff0000;">&lt;your-glue-connection-role-name&gt;</span> \
&nbsp;&nbsp; &nbsp;--policy-arn <span style="color: #ff0000;">&lt;your-glue-connection-policy-arn&gt;</span>

# Delete the custom policy
aws iam delete-policy \
&nbsp;&nbsp; &nbsp;--policy-arn <span style="color: #ff0000;">&lt;your-glue-connection-policy-arn&gt;</span>

# Delete the role
aws iam delete-role \
&nbsp;&nbsp; &nbsp;--role-name <span style="color: #ff0000;">&lt;your-glue-connection-role-name&gt;</span></code></pre> 
  </div> </li> 
 <li>Delete the Secrets Manager secret: 
  <div class="hide-language"> 
   <pre><code class="lang-code"># Schedule secret for deletion (7-30 days)
aws secretsmanager delete-secret \
&nbsp;&nbsp; &nbsp;--secret-id <span style="color: #ff0000;">&lt;your-snowflake-secret&gt;</span></code></pre> 
  </div> </li> 
</ol> 
<p>This teardown guide doesn’t affect the actual metadata in the remote catalog server nor the data in S3 buckets. It only affects the federation configurations in the Data Catalog and Lake Formation. Any corresponding service principals or configurations in the remote catalog server must be addressed separately.</p> 
<p>Make sure you follow the teardown steps in the specified order to avoid dependency conflicts. For example, an AWS Glue connection object can’t be deleted if an AWS Glue catalog object is associated with it.</p> 
<p>Additionally, make sure you have the necessary permissions to delete these resources.</p> 
<h2>Conclusion</h2> 
<p>In this post, we explored how catalog federation addresses the growing challenge of managing Iceberg tables across multi-vendor catalog environments. We walked through the architecture, demonstrating how the Data Catalog communicates with remote catalog systems, including Snowflake Polaris Catalog, Databricks Unity Catalog, and custom Iceberg REST-compliant catalogs, with centralized authorization and credential vending for secure data access. We covered the setup process, including configuring authentication principals, creating federated catalogs using AWS Glue connections, to implementing fine-grained access controls and querying remote Iceberg tables directly from AWS analytics engines.</p> 
<p>Catalog federation offers several advantages:</p> 
<ul> 
 <li>Query your Iceberg data where it lives while maintaining security, governance, and price-performance benefits of AWS analytics services</li> 
 <li>Remove operational overheads and costs to maintain synchronization processes</li> 
 <li>Avoid data duplication and inconsistencies</li> 
 <li>Get real-time access to up-to-date table schemas without migrating or replacing existing catalogs.</li> 
</ul> 
<p>To learn more, refer to <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/catalog-federation.html" rel="noopener noreferrer" target="_blank">Catalog federation to remote Iceberg catalogs</a>.</p> 
<hr /> 
<h3>About the authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Debika D" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/11/26/ddebika.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Debika D</h3> 
  <p><a href="https://www.linkedin.com/in/debikad/" rel="noopener" target="_blank">Debika</a> is a Senior Product Marketing Manager with Amazon SageMaker, specializing in messaging and go-to-market strategy for lakehouse architecture. She is passionate about all things data and AI.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Srividya Parthasarathy" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/11/26/srivipar.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Srividya Parthasarathy</h3> 
  <p><a href="https://www.linkedin.com/in/srividya-parthasarathy-8b71bb32/" rel="noopener" target="_blank">Srividya</a> is a Senior Big Data Architect on the AWS Lake Formation team. She works with the product team and customers to build robust features and solutions for their analytical data platform. She enjoys building data mesh solutions and sharing them with the community.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Pratik Das" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/11/26/pratdas.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Pratik Das</h3> 
  <p><a href="https://www.linkedin.com/in/das-pratik/" rel="noopener" target="_blank">Pratik</a> is a Senior Product Manager with AWS Lake Formation. He is passionate about all things data and works with customers to understand their requirements and build delightful experiences. He has a background in building data-driven solutions and machine learning systems.</p>
 </div> 
</footer>

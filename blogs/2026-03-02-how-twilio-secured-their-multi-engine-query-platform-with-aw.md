---
title: "How Twilio secured their multi-engine query platform with AWS Lake Formation"
url: "https://aws.amazon.com/blogs/big-data/how-twilio-secured-their-multi-engine-query-platform-with-aws-lake-formation/"
date: "Mon, 02 Mar 2026 17:49:13 +0000"
author: "Aakash Pradeep, Venkatram Bondugula"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p><em>This is a guest post by Aakash Pradeep, Principal Software Engineer, and Venkatram Bondugula, Software Engineer at Twilio, in partnership with AWS.</em></p> 
<p><a href="http://twilio.com/" rel="noopener" target="_blank">Twilio</a> is a cloud communications platform that provides programmable APIs and tools for developers to easily integrate voice, messaging, email, video, and other communication features into their applications and customer engagement workflows.</p> 
<p>In this blog series we discuss how we built a multi-engine query platform at Twilio. The <a href="https://aws.amazon.com/blogs/big-data/how-twilio-built-a-multi-engine-query-platform-using-amazon-athena-and-open-source-presto/" rel="noopener" target="_blank">first part</a> introduces the use case that led us to build a new platform and why we selected <a href="https://aws.amazon.com/athena/" rel="noopener" target="_blank">Amazon Athena</a> alongside our open-source Presto implementation. This second part discusses how Twilio’s query infrastructure platform integrates with <a href="https://aws.amazon.com/lake-formation/" rel="noopener" target="_blank">AWS Lake Formation</a> to provide fine-grained access control to all their data.</p> 
<p>At Twilio, we faced critical challenges in managing our multi-engine query platform across a complex data mesh architecture spanning multiple AWS accounts and Lines of Business. We needed a unified permissions model that could work consistently across different query engines like OSS Presto and Amazon Athena, eliminating the fragmented authentication experiences in our infrastructure. The growing demand for secure cross-account data sharing required moving beyond manual, multi-step provisioning processes that depended heavily on human intervention. Additionally, Twilio’s compliance and data stewardship requirements demanded fine-grained access controls at row, column, and cell levels, necessitating a scalable and flexible approach to permission management. By adopting the <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener" target="_blank">AWS Glue Data Catalog</a> as our managed metastore and AWS Lake Formation for governance, we implemented Tag-Based Access Control (LF-TBAC) to simplify access management, enabled data sharing through automated workflows, and established a centralized governance framework that provided uniform permissions management across all AWS services.</p> 
<h2>Transitioning to a managed metastore and governance solutions</h2> 
<p>We discussed in part 1, how we were looking to move to managed services to alleviate us of the burden of managing the underlying infrastructure of a query platform. Along with our decision to adopt Amazon Athena, we also began to evaluate the adoption of <a href="https://aws.amazon.com/emr/serverless/" rel="noopener" target="_blank">Amazon EMR Serverless</a> for our Spark workloads, which made us aware of the fact that we needed to migrate to a managed solution for our <a href="https://hive.apache.org/" rel="noopener" target="_blank">Apache Hive</a> metastore.</p> 
<p>We selected the AWS Glue Data Catalog as our managed metastore repository to support our enterprise-wide data mesh architecture. For managing permissions to the Data Catalog assets, we chose AWS Lake Formation, a service that enables data governance and security at scale using familiar database-like permissions. Lake Formation provides a unified permissions model as well as support for enabling data mesh architecture that we were seeking.</p> 
<p>Lake Formation’s support for row, column, and cell-level access controls provides the fine-grained access control (FGAC) capabilities required by our compliance and data stewardship policies. Additionally, Lake Formation’s <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/tag-based-access-control.html" rel="noopener" target="_blank">tag-based access control</a> (LF-TBAC) feature allows us to define FGAC permissions based on tags attached to the Data Catalog resources, enabling flexible and scalable permission management.</p> 
<h2>Integrating Odin with AWS Lake Formation</h2> 
<p>Odin, our Presto-based gateway, serves as a central hub for query processing, managing authentication, routing, and the complete workflow throughout a query’s lifecycle. As the primary interface, Odin enables users to connect through JDBC or APIs from various BI tools, SQL IDEs, and other applications.</p> 
<p>Beyond its core routing capabilities, Odin utilizes local caches implemented using <a href="https://github.com/google/guava" rel="noopener" target="_blank">Google’s Guava</a> caching library to optimize performance across the platform. Guava delivers efficient in-memory caching for Java applications by storing data locally within the application instance, resulting in significantly faster retrieval times. Odin employs multiple Guava caching layers across various modules to ensure optimal response times for frequently accessed data and metadata.</p> 
<p>Building on this performance foundation, Odin implements authentication and authorization layers to ensure secure and controlled access to data across multiple query engines. These security components work together to verify user identities and enforce data access policies, providing a unified security framework that abstracts away the complexities of individual engine implementations while maintaining strict governance standards.</p> 
<h3>The authentication layer</h3> 
<p>Different query engines like OSS Presto and Amazon Athena each implement their own authentication mechanisms. To create a consistent user experience, Odin provides a unified authentication layer that shields users from these underlying differences. Currently, Odin’s pluggable authentication system supports LDAP integration, with plans to expand this capability to include Okta authentication using IAM Identity center in the future.</p> 
<h3>The authorization layer</h3> 
<p>For data consumers using AWS Analytics services such as AWS Glue, Amazon EMR, and Athena through an IAM federated role-based access, AWS Lake Formation provided critical authorization capabilities for data governance through their existing integrations. However, we needed to extend its capabilities to integrate with OSS Presto. Additionally, our users for the query infrastructure platform were not mapped to an IAM user so would need to build a custom authorization layer in Odin to verify permissions and integrate with Lake Formation. Our challenge was creating a consistent way to control data access across all our query engines.</p> 
<p>When a user runs a query, Odin’s authorization layer checks three key pieces of information:</p> 
<ul> 
 <li>Table details: which database and table the query is accessing</li> 
 <li>User permissions: what data tags the user has access to</li> 
 <li>Resource tags: what security tags are attached to the requested table</li> 
</ul> 
<p>We store user permissions in <a href="https://aws.amazon.com/dynamodb/" rel="noopener" target="_blank">Amazon DynamoDB</a>, which allows us to quickly look up what each user can access. By matching the user’s tags with the table’s Lake Formation tags, we can determine if the query should be allowed. To keep things fast, we cache this information temporarily, allowing us to expedite authorization for recent requests.</p> 
<p>How the authorization works:</p> 
<ol> 
 <li>Initial check: First, we see if this user recently ran a similar successful query (within the last 5 minutes).</li> 
 <li>Gather information: We collect the table details, user permissions, and security tags—first checking our cache, then fetching from AWS Glue Data Catalog and Lake Formation if needed.</li> 
 <li>Match permissions: We compare the user’s access tags stored in a DynamoDB table against the table’s security tags in Lake Formation.</li> 
 <li>Make decision: If the user’s permissions match what’s required for their query action (like SELECT or INSERT), access is granted.</li> 
</ol> 
<p>This approach allows us to make use of Lake Formation tag-based access control while keeping our authorization logic separate from the individual query engines. By using smart caching and efficient lookups, we can verify permissions in just milliseconds.</p> 
<h2>Building a data mesh</h2> 
<p>At Twilio, we have multiple line of business (LoBs) each managing their own data platform infrastructure. The individual platforms are spread across multiple AWS accounts, and primarily store data on <a href="https://aws.amazon.com/s3/" rel="noopener" target="_blank">Amazon S3</a> in variety of open table formats, such as <a href="https://hudi.apache.org/" rel="noopener" target="_blank">Apache Hudi,</a> <a href="https://iceberg.apache.org/" rel="noopener" target="_blank">Apache Iceberg</a>, and <a href="https://delta.io/" rel="noopener" target="_blank">Delta Lake</a>. Each platform independently supports analytics and machine learning use cases, however, there was a growing need for secure sharing of data across LoBs. Additionally, we needed to enable self-service discovery and provisioning of access to the data with a centralized governance framework.</p> 
<p>Data consumers bring their own AWS accounts and choice of tools, which include not only AWS services such as Amazon Athena, <a href="https://aws.amazon.com/glue/" rel="noopener" target="_blank">AWS Glue</a> ETL jobs (Spark), and <a href="https://aws.amazon.com/emr/" rel="noopener" target="_blank">Amazon EMR</a>, but also AWS partner solutions.&nbsp;To improve the process of access fulfillment, data auditability and lowering the operational overhead involved, we needed an automated framework in place that had minimal human intervention and oversight.</p> 
<h3>Implementing a data subscription workflow</h3> 
<p>Previously, consumers requiring access to specific data sets would need to go through multiple steps to secure access, which involved several dependencies and manual actions. To simplify this process and provide a self-service capability, we decided to build a custom integration solution between <a href="https://www.servicenow.com/" rel="noopener" target="_blank">ServiceNow</a> and AWS Lake Formation. At Twilio, ServiceNow is used extensively to automate workflows and build custom applications to connect disparate systems and improve operational efficiency.</p> 
<p>We automated key parts of the data access process using Twilio’s standard tools: Git for version control, Terraform for infrastructure management, and custom scripts to execute the necessary AWS actions.</p> 
<p><strong>We automated three main use cases:</strong></p> 
<p><strong>1. Sharing data between accounts</strong></p> 
<p>When one team needs to share data with another team or with our central governance account, the process starts with a Git pull request (PR). This triggers our custom Lake Formation automation tool, which:</p> 
<ul> 
 <li>Connects to the source AWS account with admin permissions</li> 
 <li>Sets up data sharing using the security tags (LF-Tags) specified in a YAML configuration file</li> 
 <li>Completes the share using <a href="https://aws.amazon.com/ram/" rel="noopener" target="_blank">AWS Resource Access Manager</a> (RAM)</li> 
 <li>Creates resource links in the target account so the data appears in their catalog</li> 
 <li>Updates ServiceNow with the newly shared database and table information</li> 
</ul> 
<p><strong>2. Granting permissions to user roles</strong></p> 
<p>When users request access to data, our automation tool grants tag-based permissions directly to their IAM roles in Lake Formation. This happens after approval of either a Git PR or ServiceNow ticket.</p> 
<p><strong>3. Granting access to individual users</strong></p> 
<p>For individual user access requests:</p> 
<ul> 
 <li>Users submit a request in ServiceNow for specific tables</li> 
 <li>After approval, ServiceNow calls our internal API that checks relevant Lake Formation tags</li> 
 <li>The request is validated and sent to an <a href="https://aws.amazon.com/sqs/" rel="noopener" target="_blank">Amazon Simple Queue Service</a> (Amazon SQS) queue</li> 
 <li>A consumer service processes the request, updates the user’s permissions in our DynamoDB table (which Odin uses for authorization checks), and includes retry logic for reliability</li> 
 <li>Once complete, the service updates the ServiceNow ticket to notify the user</li> 
</ul> 
<p>The overall subscription and authorization flow is as shown in the diagram below:</p> 
<p><img alt="Diagram of Twilio's AWS data query platform showing user access requests flowing through ServiceNow and LF-Tag validation before queries reach Amazon Athena via Odin EC2 instances." class="alignnone wp-image-88258 size-full" height="1419" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/19/odin_auth-1-scaled.jpg" width="2560" /></p> 
<ol> 
 <li>Users submit a request in ServiceNow for access to a database, table, or LF-Tag</li> 
 <li>The system retrieves the relevant LF-Tags from Lake Formation through our API integration</li> 
 <li>Upon approval, the automation procedure adds the user to the User-To-Tag DynamoDB table, grants IAM role permissions in Lake Formation, and sets up cross-account sharing via RAM as needed</li> 
 <li>Users submit SQL query to the Odin presto gateway</li> 
 <li>Odin authorizes the user through LDAP</li> 
 <li>Odin parsers the SQL query to identify the tables involved and the action being performed (SELECT, DDL, and more)</li> 
 <li>Odin validates permissions using the User to LF-Tag mapping and Lake formation grants to authorize the SQL query based on granted permissions</li> 
 <li>If authorized, Odin routes the query to Amazon Athena or Presto</li> 
</ol> 
<blockquote>
 <p>Using standardized tools and processes to provide self-service capabilities to the users helped us scale the governance framework and support broader use cases. Important capabilities in Lake Formation, such as Tag-based access control (TBAC) and cross-account sharing of data, simplified developing automations and our overall approach to governance.</p>
</blockquote> 
<h2>Lessons learned- Cache is king</h2> 
<p><em>“By adopting AWS Glue Data Catalog as our managed metastore and AWS Lake Formation for Tag-Based Access Control, we simplified access management and enabled data sharing by reducing auth overhead to just 6-10 milliseconds through caching and targeted scaling.”</em></p> 
<p>As Odin began handling queries at scale, we encountered performance bottlenecks in our customized authorization process as we had to retrieve information from multiple services, particularly with complex queries spanning multiple tables. The authorization checks involved in the performance bottleneck frequently caused query timeouts which impacted overall system reliability. The root of the problem lay in our sequential authorization workflow: our system first had to parse each query to identify all tables requiring identity verification, then make separate API calls to the AWS Glue Data Catalog and Lake Formation for each table’s permissions. It became clear that we needed to optimize this authentication process to reduce response times and improve the overall query experience.</p> 
<p>We also recognized there were different caching needs between our POST operations and GET/DELETE HTTP calls, so we decided to separate them into two different Application Load Balancer (ALB) target groups. For POST requests, which required Lake Formation authentication, we found that concentrating traffic through just 2-3 target instances distributed across multiple Availability Zones (AZ) was more efficient. This approach allowed authentication information to be effectively cached locally on these dedicated instances, dramatically reducing the volume of API calls to the Lake Formation service.</p> 
<p>GET and DELETE requests follow a more simplified workflow. Since users have already completed initial authorization, there is no need to continue to perform authorization checks. Although they follow a simpler workflow, these requests have much higher volume with requests numbering into the 10s of millions per hour. Due to this scale, we opted to implement horizontal scaling to scale the target ALB to 10 <a href="https://aws.amazon.com/ec2/" rel="noopener" target="_blank">Amazon EC2</a> instances to fetch the query history from the DynamoDB table. These EC2 instances make use of local LRU caching with a 5-minute expiration policy for authentication data.</p> 
<p>By implementing authentication caching and adopting specialized approaches for different HTTP request types with targeted scaling groups, we successfully reduced Odin’s overall overhead to a maximum of 6-10 milliseconds for both authentication and authorization.</p> 
<h2>Conclusion and what’s next</h2> 
<p>In this post, we explored how we enhanced Odin, our unified multi-engine query platform, with authentication and authorization capabilities using AWS Lake Formation and a custom authorization workflow. By using AWS services including Lake Formation, AWS Glue Data Catalog, and Amazon DynamoDB alongside Twilio’s existing infrastructure, we created a scalable self-service governance framework that streamlines user access management, simplifies auditing, and enables seamless data sharing across our complex cloud environment. With this workflow automation, we eliminated operational overhead while building a secure, robust platform that serves as the foundation for Twilio’s data mesh architecture.</p> 
<p>Going forward,&nbsp;we are focusing on strengthening our authentication and authorization framework by enabling trusted federation with an identity provider(IdP) through <a href="https://aws.amazon.com/iam/identity-center/" rel="noopener" target="_blank">AWS IAM Identity Center</a>, which integrates directly with Lake Formation. Using Trusted Identity Propagation capabilities supported by IAM IDC will allow us to establish a consistent governance flow based on a user identity and will allow us to unlock the full capabilities of AWS Lake Formation such as&nbsp;fine-grained access control with data filters.</p> 
<p>To learn more and get started with building with AWS Lake Formation, see <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/getting-started-setup.html" rel="noopener" target="_blank">Getting started with Lake Formation</a>, and <a href="https://aws.amazon.com/blogs/big-data/build-a-modern-data-architecture-and-data-mesh-pattern-at-scale-using-aws-lake-formation-tag-based-access-control/" rel="noopener" target="_blank">How to build a data mesh architecture at scale using AWS Lake Formation tag-based access control</a>.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Aakash Pradeep" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/19/aakash-100x85.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Aakash Pradeep</h3> 
  <p><a href="https://www.linkedin.com/in/aakashpradeep/" rel="noopener" target="_blank">Aakash</a> is a Principal Software Engineer with over 15 years of experience across ingestion, compute, storage, and query platforms. Aakash is a PrestoCon speaker, holds multiple patents in real-time analytics, and is passionate about building high-performance distributed systems.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Venkatram Bondugula" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/19/venkat-100x104.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Venkatram Bondugula</h3> 
  <p><a href="https://www.linkedin.com/in/venkatram-bondugula-a3555258/" rel="noopener" target="_blank">Venkatram</a> is a seasoned backend engineer with over a decade of experience specializing in the design and development of scalable data platforms for big data and distributed systems. With a strong background in backend architecture and data engineering, he has built and optimized high-performance systems that power data-driven decision-making at scale.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Aneesh Chandra PN" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/19/aneesh-100x106.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Aneesh Chandra PN</h3> 
  <p><a href="https://www.linkedin.com/in/aneesh-chandra-pn/" rel="noopener" target="_blank">Aneesh</a> is a Principal Analytics Solutions Architect at AWS working with Strategic customers. He is passionate about using technology advancements to solve customers’ data challenges. He uses his strong expertise on analytics, distributed systems and open source frameworks to be a trusted technical advisor for AWS customers.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Amber Runnels" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/02/19/amber-100x148.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Amber Runnels</h3> 
  <p><a href="https://www.linkedin.com/in/amberrunnels/" rel="noopener" target="_blank">Amber</a> is a Senior Analytics Specialist Solutions Architect at AWS specializing in big data and distributed systems. She helps customers optimize workloads in the AWS data ecosystem to achieve a scalable, performant, and cost-effective architecture. Aside from technology, she is passionate about exploring the many places and cultures this world has to offer, reading novels, and building terrariums.</p> 
 </div> 
</footer>

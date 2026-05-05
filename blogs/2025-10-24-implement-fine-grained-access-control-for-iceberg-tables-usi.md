---
title: "Implement fine-grained access control for Iceberg tables using Amazon EMR on EKS integrated with AWS Lake Formation"
url: "https://aws.amazon.com/blogs/big-data/implement-fine-grained-access-control-for-iceberg-tables-using-amazon-emr-on-eks-integrated-with-aws-lake-formation/"
date: "Fri, 24 Oct 2025 20:39:25 +0000"
author: "Tejal Patel"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
<p>The rise of distributed data processing frameworks such as Apache Spark has revolutionized the way organizations manage and analyze large-scale data. However, as the volume and complexity of data continue to grow, the need for fine-grained access control (FGAC) has become increasingly important. This is particularly true in scenarios where sensitive or proprietary data must be shared across multiple teams or organizations, such as in the case of open data initiatives. Implementing robust access control mechanisms is crucial to maintain secure and controlled access to data stored in Open Table Format (OTF) within a modern data lake.</p> 
<p>One approach to addressing this challenge is by using <a href="https://aws.amazon.com/emr/" rel="noopener noreferrer" target="_blank">Amazon EMR</a> on <a href="https://aws.amazon.com/eks/" rel="noopener noreferrer" target="_blank">Amazon Elastic Kubernetes Service</a> (Amazon EKS) and incorporating FGAC mechanisms. With <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/emr-eks.html" rel="noopener noreferrer" target="_blank">Amazon EMR on EKS</a>, you can run open source big data frameworks such as Spark on Amazon EKS. This integration provides the scalability and flexibility of Kubernetes, while also using the data processing capabilities of Amazon EMR.</p> 
<p>On February 6<sup>th</sup> 2025, AWS introduced <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-lf-enable.html" rel="noopener noreferrer" target="_blank">fine-grained access control</a> based on <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a> for EMR on EKS from Amazon EMR 7.7 and higher version. You can now significantly enhance your data governance and security frameworks using this feature.</p> 
<p>In this post, we demonstrate how to implement FGAC on Apache Iceberg tables using EMR on EKS with Lake Formation.</p> 
<h2>Data mesh use case</h2> 
<p>With FGAC in a <a href="https://martinfowler.com/articles/data-monolith-to-mesh.html" rel="noopener noreferrer" target="_blank">data mesh</a> architecture, domain owners can manage access to their data products at a granular level. This decentralized approach allows for greater agility and control, making sure data is accessible only to authorized users and services within or across domains. Policies can be tailored to specific data products, considering factors like data sensitivity, user roles, and intended use. This localized control enhances security and compliance while supporting the self-service nature of the data mesh.</p> 
<p>FGAC is especially useful in business domains that deal with sensitive data, such as healthcare, finance, legal, human resources, and others. In this post, we focus on examples from the healthcare domain, showcasing how we can achieve the following:</p> 
<ul> 
 <li><strong>Share patient data securely</strong> – Data mesh enables different departments within a hospital to manage their own patient data as independent domains. FGAC makes sure only authorized personnel can access specific patient records or data elements based on their roles and need-to-know basis.</li> 
 <li><strong>Facilitate research and collaboration </strong>– Researchers can access de-identified patient data from various hospital domains through the data mesh architecture, enabling collaboration between multidisciplinary teams across different healthcare institutions, fostering knowledge sharing, and accelerating research and discovery. FGAC supports compliance with privacy regulations (such as HIPAA) by restricting access to sensitive data elements or allowing access only to aggregated, anonymized datasets.</li> 
 <li><strong>Improve operational efficiency </strong>– Data mesh can streamline data sharing between hospitals and insurance companies, simplifying billing and claims processing. FGAC makes sure only authorized personnel within each organization can access the necessary data, protecting sensitive financial information.</li> 
</ul> 
<h2>Solution overview</h2> 
<p>In this post, we explore how to implement FGAC on Iceberg tables within an EMR on EKS application, using the capabilities of Lake Formation. For details on how to implement FGAC on Amazon EMR on EC2, refer to <a href="https://aws.amazon.com/blogs/big-data/fine-grained-access-control-in-amazon-emr-serverless-with-aws-lake-formation/" rel="noopener noreferrer" target="_blank">Fine-grained access control in Amazon EMR Serverless with AWS Lake Formation</a>.</p> 
<p>The following components play critical roles in this solution design:</p> 
<ul> 
 <li><a href="https://iceberg.apache.org/" rel="noopener noreferrer" target="_blank">Apache Iceberg OTF</a>: 
  <ul> 
   <li>High-performance table format for large-scale analytics</li> 
   <li>Supports schema evolution, ACID transactions, and time travel</li> 
   <li>Compatible with Spark, Trino, Presto, and Flink</li> 
   <li><a href="https://aws.amazon.com/about-aws/whats-new/2024/12/amazon-s3-tables-apache-iceberg-tables-analytics-workloads/" rel="noopener noreferrer" target="_blank">Amazon S3 Tables</a> fully managed Iceberg tables for analytics workload</li> 
  </ul> </li> 
 <li><a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a>: 
  <ul> 
   <li>FGAC for data lakes</li> 
   <li>Column-, row-, and cell-level security controls</li> 
  </ul> </li> 
 <li><a href="https://aws.amazon.com/what-is/data-mesh/" rel="noopener noreferrer" target="_blank">Data mesh producers and consumers</a>: 
  <ul> 
   <li>Producers: Create and serve domain-specific data products</li> 
   <li>Consumers: Access and integrate data products</li> 
   <li>Enables self-service data consumption</li> 
  </ul> </li> 
</ul> 
<p>To demonstrate how you can use Lake Formation to implement cross-account FGAC within an EMR on EKS environment, we create tables in the <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a> in a central AWS account acting as producer and provision different user personas to reflect various roles and access levels in a separate AWS account acting as multiple consumers. Consumers can be spread across multiple accounts in real-world scenarios.</p> 
<p>The following diagram illustrates the high-level solution architecture.</p> 
<div class="wp-caption alignnone" id="attachment_84066" style="width: 1440px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-1.png"><img alt="AWS Healthcare Data Architecture: FGAC using Lake Formation Integration with EMR on EKS" class="wp-image-84066 size-full" height="559" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-1.png" style="margin: 10px 0px 10px 0px;" width="1430" /></a>
 <p class="wp-caption-text" id="caption-attachment-84066">Figure 1: High Level Solution Architecture</p>
</div> 
<p>To demonstrate the cross-account data sharing and data filtering with Lake Formation FGAC, the solution deploys two different Iceberg tables with varied access for different consumers. The permission mapping for consumers are with <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-permissions.html" rel="noopener noreferrer" target="_blank">cross-account table shares</a> and <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/data-filtering.html" rel="noopener noreferrer" target="_blank">data cell filters</a>.</p> 
<p>It has two different teams with different levels of Lake Formation permissions to access <code>Patients</code> and <code>Claims</code> Iceberg tables. The following table summarizes the solution’s user personas.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td><strong>Persona/Table Name</strong></td> 
   <td><strong>Patients</strong></td> 
   <td><strong>Claims</strong></td> 
  </tr> 
  <tr> 
   <td> <p>Patients Care Team</p> <p>(<code>team1</code> job execution role)</p></td> 
   <td> 
    <ul> 
     <li>Exclude a column <code>ssn</code></li> 
     <li>Include rows only from Texas and New York states</li> 
    </ul> </td> 
   <td>Full table access</td> 
  </tr> 
  <tr> 
   <td> <p>Claims Care Team</p> <p>(<code>team2</code> job execution role)</p></td> 
   <td>No access</td> 
   <td>Full table access</td> 
  </tr> 
 </tbody> 
</table> 
<h2>Prerequisites</h2> 
<p>This solution requires an AWS account with an <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) power user role that can create and interact with AWS services, including Amazon EMR, Amazon EKS, <a href="https://aws.amazon.com/glue" rel="noopener noreferrer" target="_blank">AWS Glue</a>, Lake Formation, and <a href="http://aws.amazon.com/s3" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3). Additional specific requirements for each account are detailed in the relevant sections.</p> 
<h2>Clone the project</h2> 
<p>To get started, download the project either to your computer or the <a href="https://aws.amazon.com/cloudshell/" rel="noopener noreferrer" target="_blank">AWS CloudShell</a> console:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">git clone https://github.com/aws-samples/sample-emr-on-eks-fgac-iceberg
 cd sample-emr-on-eks-fgac-iceberg</code></pre> 
</div> 
<h2>Set up infrastructure in producer account</h2> 
<p>To set up the infrastructure in the producer account, you must have the following additional resources:</p> 
<ul> 
 <li>The latest release version of the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI)</li> 
 <li>The latest release version of the Amazon EKS CLI (<a href="https://eksctl.io/installation/" rel="noopener noreferrer" target="_blank">eksctl</a>)</li> 
 <li>An IAM role that’s a Lake Formation administrator to run the <code>producer_iceberg_datalake_setup.sh</code> script</li> 
 <li>An S3 bucket to store <a href="https://aws.amazon.com/athena" rel="noopener noreferrer" target="_blank">Amazon Athena</a> query results</li> 
 <li>A resource policy in the Data Catalog settings to allow <a href="https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-permissions.html" rel="noopener noreferrer" target="_blank">cross-account permission grants</a></li> 
</ul> 
<p>The setup script deploys the following infrastructure:</p> 
<ul> 
 <li>An S3 bucket to store sample data in Iceberg table format, registered as a data location in Lake Formation</li> 
 <li>An AWS Glue database named <code>healthcare_db</code></li> 
 <li>Two AWS Glue tables: <code>Patients</code> and <code>Claims</code> Iceberg tables</li> 
 <li>A Lake Formation data access IAM role</li> 
 <li>Cross-account permissions enabled for the consumer account: 
  <ul> 
   <li>Allow the consumer to describe the database <code>healthcare_db</code> in the producer account</li> 
   <li>Allow to access the <code>Patients</code> table using a data cell filter, based on row-level selected <code>state</code>, and exclude column <code>ssn</code></li> 
   <li>Allow full table access to the <code>Claims</code> table</li> 
  </ul> </li> 
</ul> 
<p>Run the following <code>producer_iceberg_datalake_setup.sh</code> script to create a development environment in the producer account. Update its parameters according to your requirements:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">export AWS_REGION=us-west-2
export&nbsp;PRODUCER_AWS_ACCOUNT=&lt;YOUR_PRODUCER_AWS_ACCOUNT_ID&gt;&nbsp;
export CONSUMER_AWS_ACCOUNT=&lt;YOUR_CONSUMER_AWS_ACCOUNT_ID&gt;&nbsp;
./producer_iceberg_datalake_setup.sh&nbsp;
# run the clean-up script before re-run the setup if needed
./producer_clean_up.sh</code></pre> 
</div> 
<h2>Enable cross-account Lake Formation access in producer account</h2> 
<p>A consumer account ID and an <code>EMR on EKS Engine</code> session tag must set in the producer’s environment. It allows the consumer to access the producer’s AWS Glue tables governed by Lake Formation. Complete the following steps to enable cross-account access:</p> 
<ol> 
 <li>Open the Lake Formation console in the producer account.</li> 
 <li>Choose <strong>Application integration settings</strong> under <strong>Administration</strong> in the navigation pane.</li> 
 <li>Select <strong>Allow external engines to filter data in Amazon S3 locations registered with Lake Formation</strong>.</li> 
 <li>For <strong>Session tag values</strong>, enter <strong>EMR on EKS Engine</strong>.</li> 
 <li>For <strong>AWS account IDs</strong>, enter your consumer account ID.</li> 
 <li>Choose <strong>Save</strong>.</li> 
</ol> 
<div class="wp-caption alignnone" id="attachment_84067" style="width: 1034px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-2.jpeg"><img alt="Comprehensive AWS Lake Formation application integration settings interface for managing third-party data access." class="wp-image-84067 size-large" height="690" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-2-1024x690.jpeg" style="margin: 10px 0px 10px 0px;" width="1024" /></a>
 <p class="wp-caption-text" id="caption-attachment-84067">Figure 2: Producer Account – Lake Formation third-party engine configuration screen with session tags, account IDs, and data access permissions.</p>
</div> 
<h2>Validate FGAC setup in producer environment</h2> 
<p>To validate the FGAC setup in the producer account, check the Iceberg tables, data filter, and FGAC permission settings.</p> 
<h3>Iceberg tables</h3> 
<p>Two AWS Glue tables in Iceberg format were created by <code>producer_iceberg_datalake_setup.sh</code>. On the Lake Formation console, choose <strong>Tables </strong>under <strong>Data Catalog </strong>in the navigation pane to see the tables listed.</p> 
<div class="wp-caption alignnone" id="attachment_84068" style="width: 1437px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-3.png"><img alt="AWS Lake Formation Tables interface showing a success message for updated external data filtering settings, with a table list displaying healthcare database tables in Apache Iceberg format." class="wp-image-84068 size-full" height="493" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-3.png" style="margin: 10px 0px 10px 0px;" width="1427" /></a>
 <p class="wp-caption-text" id="caption-attachment-84068">Figure 3: Lake Formation interface displaying claims and patients tables from healthcare_db with Apache Iceberg format.</p>
</div> 
<p>The following screenshot shows an example of the <code>patients</code> table data.</p> 
<div class="wp-caption alignnone" id="attachment_84069" style="width: 1400px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-4.png"><img alt="Patients table data" class="wp-image-84069 size-full" height="396" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-4.png" style="margin: 10px 0px 10px 0px;" width="1390" /></a>
 <p class="wp-caption-text" id="caption-attachment-84069">Figure 4: Patients table data</p>
</div> 
<p>The following screenshot shows an example of the <code>claims</code> table data.</p> 
<div class="wp-caption alignnone" id="attachment_84070" style="width: 1299px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-5.jpeg"><img alt="claims table data" class="wp-image-84070 size-full" height="373" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-5.jpeg" style="margin: 10px 0px 10px 0px;" width="1289" /></a>
 <p class="wp-caption-text" id="caption-attachment-84070">Figure 5: Claims table data</p>
</div> 
<h3>Data cell filter against patients table</h3> 
<p>After successfully running the <code>producer_iceberg_datalake_setup.sh</code> script, a new data cell filter named <code>patients_column_row_filter</code> was created in Lake Formation. This filter performs two functions:</p> 
<ul> 
 <li>Exclude the <code>ssn</code> column from the <code>patients</code> table data</li> 
 <li>Include rows where the state is Texas or New York</li> 
</ul> 
<p>To view the data cell filter, choose <strong>Data filters</strong> under <strong>Data Catalog</strong> in the navigation pane of the Lake Formation console, and open the filter. Choose <strong>View permission </strong>to view the permission details.</p> 
<div class="wp-caption alignnone" id="attachment_84071" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-6.png"><img alt="Data cell filter" class="wp-image-84071 size-full" height="740" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-6.png" style="margin: 10px 0px 10px 0px;" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-84071">Figure 6: Column and Row level filter configuration for patients table</p>
</div> 
<h3>FGAC permissions allowing cross-account access</h3> 
<p>To view all the FGAC permissions, choose <strong>Data permissions </strong>under <strong>Permissions </strong>in the navigation pane of the Lake Formation console, and filter by the database name <code>healthcare_db</code>.</p> 
<p>Make sure to revoke data permissions with the <code>IAMAllowedPrincipals</code> principal associated to the <code>healthcare_db</code> tables, because it will cause cross-account data sharing to fail, particularly with <a href="http://aws.amazon.com/ram" rel="noopener noreferrer" target="_blank">AWS Resource Access Manager</a> (AWS RAM).</p> 
<div class="wp-caption alignnone" id="attachment_84078" style="width: 1441px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-7.png"><img alt="Data permissions overview" class="wp-image-84078 size-full" height="385" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-7.png" style="margin: 10px 0px 10px 0px;" width="1431" /></a>
 <p class="wp-caption-text" id="caption-attachment-84078">Figure 7: Lake Formation data permissions interface displaying filtered healthcare database resources with granular access controls</p>
</div> 
<p>The following table summarizes the overall FGAC setup.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td><strong>Resource Type</strong></td> 
   <td><strong>Resource</strong></td> 
   <td><strong>Permissions</strong></td> 
   <td><strong>Grant Permissions</strong></td> 
  </tr> 
  <tr> 
   <td>Database</td> 
   <td> 
    <div class="hide-language"> 
     <pre><code class="lang-code">healthcare_db</code></pre> 
    </div> </td> 
   <td>Describe</td> 
   <td>Describe</td> 
  </tr> 
  <tr> 
   <td>Data Cell Filter</td> 
   <td> 
    <div class="hide-language"> 
     <pre><code class="lang-code">patients_column_row_filter</code></pre> 
    </div> </td> 
   <td>Select</td> 
   <td>Select</td> 
  </tr> 
  <tr> 
   <td>Table</td> 
   <td> 
    <div class="hide-language"> 
     <pre><code class="lang-code">Claims</code></pre> 
    </div> </td> 
   <td>Select, Describe</td> 
   <td>Select, Describe</td> 
  </tr> 
 </tbody> 
</table> 
<h2>Set up infrastructure in consumer account</h2> 
<p>To set up the infrastructure in the consumer account, you must have the following additional resources:</p> 
<ul> 
 <li><a href="https://eksctl.io/installation/" rel="noopener noreferrer" target="_blank">eksctl</a> and <a href="https://kubernetes.io/docs/tasks/tools/" rel="noopener noreferrer" target="_blank">kubectl</a> packages must be installed</li> 
 <li>An IAM role in the consumer account must be a Lake Formation administrator to run <code>consumer_emr_on_eks_setup.sh</code> script</li> 
 <li>The Lake Formation admin must accept the <a href="https://console.aws.amazon.com/ram/home?#SharedResourceShares" rel="noopener noreferrer" target="_blank">AWS RAM resource share</a> invites using the AWS RAM console, if the consumer account is outside of the producer’s organizational unit</li> 
</ul> 
<div class="wp-caption alignnone" id="attachment_84079" style="width: 1298px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-8.jpeg"><img alt="RAM resource share screen " class="wp-image-84079 size-full" height="315" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-8.jpeg" style="margin: 10px 0px 10px 0px;" width="1288" /></a>
 <p class="wp-caption-text" id="caption-attachment-84079">Figure 8: Consumer account – Cross-account RAM share for Lake Formation resource</p>
</div> 
<p>The setup script deploys the following infrastructure:</p> 
<ul> 
 <li>An EKS cluster called <code>fgac-blog</code> with two namespaces: 
  <ul> 
   <li>User namespace: <code>lf-fgac-user</code></li> 
   <li>System namespace:<code>lf-fgac-secure</code></li> 
  </ul> </li> 
 <li>An EMR on EKS virtual cluster <code>emr-on-eks-fgac-blog</code>: 
  <ul> 
   <li>Set up with a security configuration <code>emr-on-eks-fgac-sec-conifg</code></li> 
   <li>Two EMR on EKS job execution IAM roles: 
    <ul> 
     <li>Role for the Patients Care Team (<code>team1</code>): <code>emr_on_eks_fgac_job_team1_execution_role</code></li> 
     <li>Role for Claims Care Team (<code>team2</code>): <code>emr_on_eks_fgac_job_team2_execution_role</code></li> 
    </ul> </li> 
   <li>A <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-lf-enable.html#security_iam_fgac-lf-system-profile-configure" rel="noopener noreferrer" target="_blank">query engine IAM role</a> used by FGAC secure space: <code>emr_on_eks_fgac_query_execution_role</code></li> 
  </ul> </li> 
 <li>An S3 bucket to store PySpark job scripts and logs</li> 
 <li>An AWS Glue local database named <code>consumer_healthcare_db</code></li> 
 <li>Two resource links to cross-account shared AWS Glue tables: <code>rl_patients</code> and <code>rl_claims</code></li> 
 <li>Lake Formation permission on Amazon EMR IAM roles</li> 
</ul> 
<p>Run the following <a href="https://gitlab.aws.dev/thpatel/emr-on-eks-fgac-iceberg/-/blob/dev/consumer_account_setup/consumer_emr_on_eks_setup.sh?ref_type=heads" rel="noopener noreferrer" target="_blank">consumer_emr_on_eks_setup.sh</a> script to set up a development environment in the consumer account. Update the parameters according to your use case:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">export AWS_REGION=us-west-2&nbsp;
export&nbsp;PRODUCER_AWS_ACCOUNT=&lt;YOUR_PRODUCER_AWS_ACCOUNT_ID&gt;&nbsp;
export EKSCLUSTER_NAME=fgac-blog&nbsp;
./consumer_emr_on_eks_setup.sh&nbsp;
# run the clean-up script before re-run the setup if needed
./consumer_clean_up.sh</code></pre> 
</div> 
<h2>Enable cross-account Lake Formation access in consumer account</h2> 
<p>The consumer account must add the consumer account ID with an <code>EMR on EKS Engine</code> session tag in Lake Formation. This session tag will be used by EMR on EKS job execution IAM roles to access Lake Formation tables. Complete the following steps:</p> 
<ol> 
 <li>Open the Lake Formation console in the consumer account.</li> 
 <li>Choose <strong>Application integration settings</strong> under <strong>Administration</strong> in the navigation pane.</li> 
 <li>Select <strong>Allow external engines to filter data in Amazon S3 locations registered with Lake Formation</strong>.</li> 
 <li>For <strong>Session tag values</strong>, enter <strong>EMR on EKS Engine</strong>.</li> 
 <li>For <strong>AWS account IDs</strong>, enter your consumer account ID.</li> 
 <li>Choose <strong>Save</strong>.</li> 
</ol> 
<div class="wp-caption alignnone" id="attachment_84080" style="width: 1126px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-9.jpeg"><img alt="" class="size-full wp-image-84080" height="752" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-9.jpeg" style="margin: 10px 0px 10px 0px;" width="1116" /></a>
 <p class="wp-caption-text" id="caption-attachment-84080">Figure 9: Consumer Account – Lake Formation third-party engine configuration screen with session tags, account IDs, and data access permissions</p>
</div> 
<h2>Validate FGAC setup in consumer environment</h2> 
<p>To validate the FGAC setup in the producer account, check the EKS cluster, namespaces, and Spark job scripts to test data permissions.</p> 
<h3>EKS cluster</h3> 
<p>On the Amazon EKS console, choose <strong>Clusters</strong> in the navigation pane and confirm the EKS cluster <code>fgac-blog</code> is listed.</p> 
<div class="wp-caption alignnone" id="attachment_84081" style="width: 1907px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-10.png"><img alt="EKS Cluster view page" class="wp-image-84081 size-full" height="754" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-10.png" style="margin: 10px 0px 10px 0px;" width="1897" /></a>
 <p class="wp-caption-text" id="caption-attachment-84081">Figure 10: Consumer Account – EKS Cluster console page</p>
</div> 
<h3>Namespaces in Amazon EKS</h3> 
<p>Kubernetes uses namespaces as logical partitioning system for organizing objects such as Pods and Deployments. Namespaces also operate as a privilege boundary in the Kubernetes role-based access control (RBAC) system. Multi-tenant workloads in Amazon EKS can be secured using <a href="https://docs.aws.amazon.com/whitepapers/latest/security-practices-multi-tenant-saas-applications-eks/use-namespaces-to-separate-tenant-workloads.html" rel="noopener noreferrer" target="_blank">namespaces</a>.</p> 
<p>This solution creates two namespaces:</p> 
<ul> 
 <li><code>lf-fgac-user</code></li> 
 <li><code>lf-fgac-secure</code></li> 
</ul> 
<p>The <code>StartJobRun</code> API uses the backend workflows to submit a Spark job’s <code>UserComponents</code> (<code>JobRunner</code>, <code>Driver</code>, <code>Executors</code>) in the user namespace, and the corresponding system components in the system namespace to accomplish the desired FGAC behaviors.</p> 
<p>You can verify the namespaces with the following command:<code>kubectl get namespace</code>The following screenshot shows an example of the expected output.</p> 
<div class="wp-caption alignnone" id="attachment_84082" style="width: 1128px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-11.png"><img alt="Namespace summary page" class="wp-image-84082 size-full" height="226" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-11.png" style="margin: 10px 0px 10px 0px;" width="1118" /></a>
 <p class="wp-caption-text" id="caption-attachment-84082">Figure 11: EKS Cluster namespaces</p>
</div> 
<h3>Spark job script to test Patients Care Team’s data permissions</h3> 
<p>Starting with Amazon EMR version 6.6.0, you can use Spark on EMR on EKS with the Iceberg table format. For more information on how Iceberg works in an immutable data lake, see <a href="https://aws.amazon.com/blogs/big-data/build-a-high-performance-acid-compliant-evolving-data-lake-using-apache-iceberg-on-amazon-emr/" rel="noopener noreferrer" target="_blank">Build a high-performance, ACID compliant, evolving data lake using Apache Iceberg on Amazon EMR</a>.</p> 
<p>The following script is a snippet of the <a href="https://github.com/aws-samples/sample-emr-on-eks-fgac-iceberg/blob/main/consumer_account_setup/consumer_emr_on_eks_setup.sh?ref_type=heads#L648" rel="noopener noreferrer" target="_blank">PySpark job</a> that retrieves filtered data for the <code>Claims</code> and <code>Patient</code> tables:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">    print("Patient Care Team PySpark job running on EMR on EKS! to query Patients and Claims tables!")
    print("This job queries Patients and Claims tables!")
    df1 = spark.sql('SELECT * FROM dev.${CONSUMER_DATABASE}.${rl_patients}')
    print("Patients tables data:")
    print("Note: Patients table is filtered on SSN column and it shows records only for Texas and New York states")
    df1.show(20)
    df2 = spark.sql('SELECT p.state,
                            c.claim_id,
                            c.claim_date, 
                            p.patient_name, 
                            c.diagnosis_code, 
                            c.procedure_code, 
                            c.amount, 
                            c.status, 
                            c.provider_id 
                    FROM dev.${CONSUMER_DATABASE}.${rl_claims} c 
                    JOIN dev.${CONSUMER_DATABASE}.${rl_patients} p
                   ON c.patient_id = p.patient_id 
                   ORDER BY p.state, c.claim_date')
    print("Show only relevant Claims data for Patients selected from Texas and New York state:")
    df2.show(20)
    print("Job Complete")
....	</code></pre> 
</div> 
<h3>Spark job script to test Claims Care Team’s data permissions</h3> 
<p>The following script is a snippet of the <a href="https://github.com/aws-samples/sample-emr-on-eks-fgac-iceberg/blob/main/consumer_account_setup/consumer_emr_on_eks_setup.sh?ref_type=heads#L718" rel="noopener noreferrer" target="_blank">PySpark job</a> that retrieves data from the <code>Claims</code> table:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">&nbsp;&nbsp; &nbsp;print("Claims Team PySpark job running on EMR on EKS to query Claims table!")
    print("Note: Claims Team has full access to Claims table!")
    df = spark.sql('SELECT * FROM     dev.${CONSUMER_DATABASE}.${rl_claims}')
    df.show(20)
....</code></pre> 
</div> 
<h2>Validate job execution roles for EMR on EKS</h2> 
<p>The Patients Care Team uses the <code>emr_on_eks_fgac_job_team1_execution_role</code> IAM role to execute a PySpark job on EMR on EKS. The job execution role has permission to query both the <code>Patients</code> and <code>Claims</code> tables.</p> 
<p>The Claims Care Team uses the <code>emr_on_eks_fgac_job_team2_execution_role</code> IAM role to execute jobs on EMR on EKS. The job execution role only has permission to access <code>Claims</code> data.</p> 
<p>Both IAM job execution roles have the following permissions:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "EmrGetCertificate",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "emr-containers:CreateCertificate",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "LakeFormationManagedAccess",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"lakeformation:GetDataAccess",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:GetTable",
                "glue:GetCatalog",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:Create*",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"glue:Update*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "EmrSparkJobAccess",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:s3:::${S3_BUCKET}*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<p>The following code is the job execution IAM role trust policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "TrustQueryEngineRoleToAssume",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"AWS": "arn:aws:iam::$CONSUMER_ACCOUNT:role/$query_engine_role"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"sts:AssumeRole",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"sts:TagSession"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"StringLike": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"aws:RequestTag/LakeFormationAuthorizedCaller": "EMR on EKS Engine"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "TrustQueryEngineRoleToAssumeRoleOnly",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"AWS": "arn:aws:iam::$CONSUMER_ACCOUNT:role/$query_engine_role"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "sts:AssumeRole"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Federated": "arn:aws:iam::$CONSUMER_ACCOUNT oidc-provider/oidc.eks.$AWS_REGION.amazonaws.com/id/xxxxx"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "sts:AssumeRoleWithWebIdentity",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"StringLike": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"oidc.eks.$AWS_REGION.amazonaws.com/id/xxxxx:sub": "system:serviceaccount:lf-fgac-user:emr-containers-sa-*-*-$CONSUMER_ACCOUNT-&lt;hash36ofiamrole&gt;"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<p>The following code is the query engine IAM role policy (<code>emr_on_eks_fgac_query_execution_role-policy</code>):</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "AssumeJobExecutionRole",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"sts:AssumeRole",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"sts:TagSession"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource":&nbsp;["arn:aws:iam::$CONSUMER_ACCOUNT:role/emr_on_eks_fgac_job_team1_execution_role",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:iam::$CONSUMER_ACCOUNT:role/emr_on_eks_fgac_job_team2_execution_role"],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"StringLike": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"aws:RequestTag/LakeFormationAuthorizedCaller": "EMR on EKS Engine"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "AssumeJobExecutionRoleOnly",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"sts:AssumeRole"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:iam::$CONSUMER_ACCOUNT:role/emr_on_eks_fgac_job_team1_execution_role",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:iam::$CONSUMER_ACCOUNT:role/emr_on_eks_fgac_job_team2_execution_role"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<p>The following code is the query engine IAM role trust policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"AWS": "arn:aws:iam::$CONSUMER_ACCOUNT:root"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "sts:AssumeRole",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Federated": "arn:aws:iam::$CONSUMER_ACCOUNT:oidc-provider/xxxxx"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "sts:AssumeRoleWithWebIdentity",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"StringLike": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"xxxxxx:sub": "system:serviceaccount:lf-fgac-secure:emr-containers-sa-*-*-$CONSUMER_ACCOUNT-&lt;hash36ofiamrole&gt;"
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<h2>Run PySpark jobs on EMR on EKS with FGAC</h2> 
<p>For more details about how to work with Iceberg tables in EMR on EKS jobs, refer to <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/tutorial-iceberg.html" rel="noopener noreferrer" target="_blank">Using Apache Iceberg with Amazon EMR on EKS</a>. Complete the following steps to run the PySpark jobs on EMR on EKS with FGAC:</p> 
<ol> 
 <li>Run the following commands to run the patients and claims jobs:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">bash /tmp/submit-patients-job.sh
bash /tmp/submit-claims-job.sh</code></pre> 
</div> 
<ol start="2"> 
 <li>Watch the application logs from the Spark driver pod:</li> 
</ol> 
<p><code>kubectl logs drive-pod-name -c spark-kubernetes-driver -n lf-fgac-user -f</code></p> 
<p>Alternatively, you can navigate to the Amazon EMR console, open your virtual cluster, and choose the open icon next to the job to open the Spark UI and monitor the job progress.</p> 
<div class="wp-caption alignnone" id="attachment_84083" style="width: 1652px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-12.png"><img alt="Spark UI navigation" class="wp-image-84083 size-full" height="183" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-12.png" style="margin: 10px 0px 10px 0px;" width="1642" /></a>
 <p class="wp-caption-text" id="caption-attachment-84083">Figure 12: EMR on EKS job runs</p>
</div> 
<h2>View PySpark jobs output on EMR on EKS with FGAC</h2> 
<p>In Amazon S3, navigate to the Spark output logs folder:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">s3://blog-emr-eks-fgac-test-&lt;acct-id&gt;-us-west-2-dev/spark-logs/&lt;emr-on-eks-cluster-id&gt;/jobs/&lt;patients-job-id&gt;/containers/spark-xxxxxx/spark-xxxxx-driver/stdout.gz</code></pre> 
</div> 
<div class="wp-caption alignnone" id="attachment_84084" style="width: 1666px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-13.png"><img alt="S3 path to view logs" class="wp-image-84084 size-full" height="570" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-13.png" style="margin: 10px 0px 10px 0px;" width="1656" /></a>
 <p class="wp-caption-text" id="caption-attachment-84084">Figure 13: EMR on EKS job’s stdout.gz location on S3 Bucket</p>
</div> 
<p>The Patients Care Team PySpark job has query access to the <code>Patients</code> and <code>Claims</code> tables. The <code>Patients</code> table has filtered out the <code>SSN</code> column and only shows records for Texas and New York claim records, as specified in our FGAC setup.</p> 
<p>The following screenshot shows the <code>Claims</code> table for only Texas and New York.</p> 
<div class="wp-caption alignnone" id="attachment_84085" style="width: 819px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-14.png"><img alt="Claims data in consumer view" class="wp-image-84085 size-full" height="174" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-14.png" style="margin: 10px 0px 10px 0px;" width="809" /></a>
 <p class="wp-caption-text" id="caption-attachment-84085">Figure 14: EMR on EKS Spark job output</p>
</div> 
<p>The following screenshot shows the <code>Patients</code> table without the <code>SSN</code> column.</p> 
<div class="wp-caption alignnone" id="attachment_84086" style="width: 884px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-15.png"><img alt="Patients data in consumer view" class="wp-image-84086 size-full" height="207" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-15.png" style="margin: 10px 0px 10px 0px;" width="874" /></a>
 <p class="wp-caption-text" id="caption-attachment-84086">Figure 15: EMR on EKS Spark job output</p>
</div> 
<p>Similarly, navigate to the Spark output log folder for the Claims Care Team job:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">s3://blog-emr-eks-fgac-test-&lt;acct-id&gt;-us-west-2-dev/spark-logs/&lt;emr-on-eks-cluster-id&gt;/jobs/&lt;claims-job-id&gt;/containers/spark-xxxxxx/spark-xxxxx-driver/stdout.gz</code></pre> 
</div> 
<p>As shown in the following screenshot, the Claims Care Team only has access to the <code>Claims</code> table, so when the job tried to access the <code>Patients</code> table, it received an access denied error.</p> 
<div class="wp-caption alignnone" id="attachment_84087" style="width: 1052px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/03/bdb3741-16.png"><img alt="Access denied for Claims team" class="wp-image-84087 size-full" height="260" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/bdb3741-16.png" style="margin: 10px 0px 10px 0px;" width="1042" /></a>
 <p class="wp-caption-text" id="caption-attachment-84087">Figure 16: EMR on EKS Spark job output</p>
</div> 
<h2>Considerations and limitations</h2> 
<p>Although the approach discussed in this post provides valuable insights and practical implementation strategies, it’s important to recognize the key <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-considerations.html" rel="noopener noreferrer" target="_blank">considerations and limitations</a> before you start using this feature. To learn more about using EMR on EKS with Lake Formation, refer to <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-lf-works.html" rel="noopener noreferrer" target="_blank">How Amazon EMR on EKS works with AWS Lake Formation</a>.</p> 
<h2>Clean up</h2> 
<p>To avoid incurring future charges, delete the resources generated if you don’t need the solution anymore. Run the following cleanup scripts (change the AWS Region if necessary).Run the following script in the consumer account:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">export AWS_REGION=us-west-2
export PRODUCER_AWS_ACCOUNT=&lt;YOUR_PRODUCER_AWS_ACCOUNT_ID&gt;
export EKSCLUSTER_NAME=fgac-blog
./consumer_clean_up.sh</code></pre> 
</div> 
<p>Run the following script in the producer account:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">export AWS_REGION=us-west-2
export PRODUCER_AWS_ACCOUNT=&lt;YOUR_PRODUCER_AWS_ACCOUNT_ID&gt;
export CONSUMER_AWS_ACCOUNT=&lt;YOUR_CONSUMER_AWS_ACCOUNT_ID&gt;
./producer_clean_up.sh</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to integrate Lake Formation with EMR on EKS to implement fine-grained access control on Iceberg tables. This integration offers organizations a modern approach to enforcing detailed data permissions within a multi-account open data lake environment. By centralizing data management in a primary account and carefully regulating user access in secondary accounts, this strategy can simplify governance and enhance security.</p> 
<p>For more information about Amazon EMR 7.7 in reference to EMR on EKS, see <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/emr-eks-7.7.0.html" rel="noopener noreferrer" target="_blank">Amazon EMR on EKS 7.7.0 releases</a>. To learn more about using Lake Formation with EMR on EKS, see <a href="https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-lf-enable.html" rel="noopener noreferrer" target="_blank">Enable Lake Formation with Amazon EMR on EKS</a>.</p> 
<p>We encourage you to explore this solution for your specific use cases and share your feedback and questions in the comments section.</p> 
<hr /> 
<h3>About the authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Janakiraman Shanmugam" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/jashanmu.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Janakiraman Shanmugam</h3> 
  <p><a href="https://www.linkedin.com/in/janakiraman-shanmugam/" rel="noopener" target="_blank">Janakiraman</a> is a Senior Data Architect at Amazon Web Services . He has a focus in Data &amp; Analytics and enjoys helping customers to solve Big data &amp; machine learning problems.&nbsp;Outside of the office, he loves to be with his friends and family and spend time outdoors.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Tejal Patel" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/internal-cdn.amazon.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Tejal Patel</h3> 
  <p>Tejal is Sr. Delivery Consultant from AWS Professional Services team, specializing in Data Analytics and ML solutions. She helps customers design scalable and innovative solutions with the AWS Cloud. Outside of her professional life, Tejal enjoys spending time with her family and friends.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Prabhakaran Thatchinamoorthy" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2025/10/07/PrabhakaranThatchinamoorthy.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Prabhakaran Thatchinamoorthy</h3> 
  <p><a href="https://www.linkedin.com/in/prabhakaranthatchinamoorthy/" rel="noopener" target="_blank">Prabhakaran</a> is a Software Engineer at Amazon Web Services, working on the EMR on EKS service. He specializes in building and operating multi-tenant data processing platforms on Kubernetes at scale. His areas of interest include open-source batch and streaming frameworks, data tooling, and DataOps.</p> 
 </div> 
</footer>

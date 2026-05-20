---
title: "Access Databricks Unity Catalog data using catalog federation in the AWS Glue Data Catalog"
url: "https://aws.amazon.com/blogs/big-data/access-databricks-unity-catalog-data-using-catalog-federation-in-the-aws-glue-data-catalog/"
date: "Mon, 12 Jan 2026 20:37:44 +0000"
author: "Srividya Parthasarathy"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
AWS has launched the catalog federation capability, enabling direct access to Apache Iceberg tables managed in Databricks Unity Catalog through the AWS Glue Data Catalog. With this integration, you can discover and query Unity Catalog data in Iceberg format using an Iceberg REST API endpoint, while maintaining granular access controls through AWS Lake Formation. In this post, we demonstrate how to set up catalog federation between the Glue Data Catalog and Databricks Unity Catalog, enabling data querying using AWS analytics services.

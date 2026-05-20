---
title: "Access Snowflake Horizon Catalog data using catalog federation in the AWS Glue Data Catalog"
url: "https://aws.amazon.com/blogs/big-data/access-snowflake-horizon-catalog-data-using-catalog-federation-in-the-aws-glue-data-catalog/"
date: "Wed, 14 Jan 2026 21:00:26 +0000"
author: "Andries Engelbrecht"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
AWS has introduced a new catalog federation feature that enables direct access to Snowflake Horizon Catalog data through AWS Glue Data Catalog. This integration allows organizations to discover and query data in Iceberg format while maintaining security through AWS Lake Formation. This post provides a step-by-step guide to establishing this integration, including configuring Snowflake Horizon Catalog, setting up authentication, creating necessary IAM roles, and implementing AWS Lake Formation permissions.

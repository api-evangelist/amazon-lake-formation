---
title: "Configure cross-account access of Amazon SageMaker Lakehouse multi-catalog tables using AWS Glue 5.0 Spark"
url: "https://aws.amazon.com/blogs/big-data/configure-cross-account-access-of-amazon-sagemaker-lakehouse-multi-catalog-tables-using-aws-glue-5-0-spark/"
date: "Fri, 09 May 2025 17:18:44 +0000"
author: "Aarthi Srinivasan"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
In this post, we show you how to share an Amazon Redshift table and Amazon S3 based Iceberg table from the account that owns the data to another account that consumes the data. In the recipient account, we run a join query on the shared data lake and data warehouse tables using Spark in AWS Glue 5.0. We walk you through the complete cross-account setup and provide the Spark configuration in a Python notebook.

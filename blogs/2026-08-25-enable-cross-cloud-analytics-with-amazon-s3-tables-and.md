---
title: "Enable cross-cloud analytics with Amazon S3 Tables and Google BigQuery, Part 2: access control with Lake Formation"
url: "https://aws.amazon.com/blogs/big-data/enable-cross-cloud-analytics-with-amazon-s3-tables-and-google-bigquery-part-2-access-control-with-lake-formation/"
date: "2026-08-25"
author: "Lakshmi Nair"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
In Part 2 of this series, connect Google BigQuery to Amazon S3 Tables using AWS Lake Formation credential vending. Lake Formation manages fine-grained permissions and issues short-lived, scoped credentials to external engines, so you can centrally govern which teams and query engines read your Iceberg tables on AWS without managing IAM policies for every consumer.

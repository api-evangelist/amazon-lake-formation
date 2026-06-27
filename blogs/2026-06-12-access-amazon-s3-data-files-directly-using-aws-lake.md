---
title: "Access Amazon S3 data files directly using AWS Lake Formation permissions"
url: "https://aws.amazon.com/blogs/big-data/access-amazon-s3-data-files-directly-using-aws-lake-formation-permissions/"
date: "2026-06-12"
author: "Aarthi Srinivasan"
feed_url: "https://aws.amazon.com/blogs/big-data/category/analytics/aws-lake-formation/feed/"
---
Demonstrates how to read from and write to Lake Formation-managed S3 locations using Apache Spark jobs from EMR. The post explains the new GetTemporaryDataLocationCredentials() API, which provides temporary credentials scoped to registered S3 locations. It shows how data scientists can access both structured tables through SQL queries and underlying data files through programmatic APIs, with unified governance through Lake Formation permissions across AWS services and tools.

---
layout: post
title: Evaluating CloudDatastore
date:   2017-09-21 21:57:00 +0100
categories: Blu Cloud Datastore Full-text search dotnetcore
---

Now as I successfully integrated my dotnetcore API with Cloud Datastore I wanted to give it a spin, and evaluate it in terms of:
* Performance (entity retrieval speed)
* Filtering capabilities / Full-text search
* Integration with 3rdparty products (indexers, search enginges)
* Backup / Restore

# Performance

Insert Speed (not relevant)

Query Speed
Test results with 4 entities:
avg 100ms
Test results with 5k entities:
avg 40-190ms

# Filtering / Full-text search
No available.
Hack: use search tags as string array

# Integration with 3rdParty
ElasticSearch integration in Gcloud

# Backup / Restore
Backup to Google Storage Bucket




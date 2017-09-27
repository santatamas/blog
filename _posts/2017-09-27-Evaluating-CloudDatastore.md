---
layout: post
title: Evaluating CloudDatastore
date:   2017-09-21 21:57:00 +0100
categories: Blu Cloud Datastore Full-text search dotnetcore
---

As I successfully integrated my dotnetcore API with Cloud Datastore I wanted to give it a spin, and evaluate it in terms of:
* Performance (entity retrieval speed)
* Filtering capabilities / Full-text search
* Backup / Restore

# Performance
Cloud Datastore is advertised as:
``` Massive scalability with high performance. Cloud Datastore uses a distributed architecture to automatically manage scaling. Cloud Datastore uses a mix of indexes and query constraints so your queries scale with the size of your result set, not the size of your data set.```

I've decided to give it a try, and test this statement. I've seeded the datastore with ~9000 entities, then ran a set of predefined queries, with pseudo-random search terms (which all had a guaranteed number of results).
Here are the results:

![](/images/articles/testresult.png "Query test result")
*(The average time is calculated from 10 consecutive runs.)*

As you can see, the execution time has a linear correlation with the number of returned entities. It makes sense, as I think the majority of the execution time is probably spent on serialising and de-serialising the result set.
I have to mention that I ran the tests at home, and even though I have pretty good internet connection, it still renders the execution time irrelevant. If you're interested I can run the same tests in my Kubernetes cluster to get a more realistic testresult.

Personally, I'm quite satisfied with the performance, especially considering that it's free.

What I found interesting is the sheer amount of index it generates to the dataset *by default*.
![alt text](/images/articles/datastore.png "Cloud Datastore Dashboard")

As you can see, the size of the indexes is **6 times** the size of the actual data.
And I didn't even specified custom indexes yet...worth keeping this in mind!

# Filtering / Full-text search
Querying the datastore is a really straightforward process, and it's also [heavily documented here](https://cloud.google.com/datastore/docs/concepts/queries#filters)

What was a bit disappointing though is that fact that just like most of the mainstream NoSql services, Cloud Datastore also lacks full-text search capabilities.
Google's recommendation is to spin up an ElasticSearch VM or cluster, and use it for indexing and searching the DB. In early 2016, Eastic Cloud has launched on GCloud officially, which means that now you can deploy a preconfigured ElasticSearch cluster with 1 click.

Fair enough, but that would require another VM instance (or more, the default setup costs you a few hundred dollars a month!), and I'm aiming for the cheapest - but still viable - solution.

What can we do then? I did what's considered the standard fall-back solution in this case. I've created a SearchTags (string[]) field on my entity, which contains the words from all the other searchable fields. Now I can match for exact words in my texts.
I could tweak this by inserting word fragments into the search tags, but for now I guess it's good-enough for my purposes. Obviously, I don't consider this a proper and long-term solution.

# Backup / Restore
Doing backups and restores are surprisingly easy.

First, you have to enable the Admin application on the Cloud Console:
![alt text](/images/articles/console_admin.png "Admin section")

Once you've done that, it spins up a python app for you using the App Engine, which will run the Cloud Datastore admin application. I'm not sure why do we need 2 separate UIs for administration, but hey.

From there, you can back up your entities to a Google Cloud Storage bucket.
![alt text](/images/articles/backup.png "Admin application - Backup screen")

The restore procedure is similar, just enter the path to your bucket, and click import.
It certeainly doesn't *feel* very sophisticated, but it's still *waay* better than DynamoDb in this regard.

# Conclusion
Cloud Datastore definitely has it's drawbacks, and you WILL have to make compromises along the way. But the price / availability / performance ratio is *soo good* that it's very hard to say no. I'll stick to it for the next few months as it won't cost me a dime, but it's highly possible that I'll have to swap it to a more feature-rich DB engine in the future.








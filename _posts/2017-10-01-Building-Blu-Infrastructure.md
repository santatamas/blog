---
title: Building a social media site - Infrastructure
date:   2017-10-01 11:58:00 +0100
categories: GCP Cloud Kubernetes dotnetcore CDN DevOps Infrastructure
---

In the past few days I've been working on setting up the infrastructure for my cloud webapp. I chose Google Cloud, because I had no previous experience with it. Sounds weird eh? You should always choose the right tools for the right job, but in this case it's also a learning experience for me.

I've been mostly involved in projects using AWS, so I have a fairly good understanding of its capabilities. Hence I thought this is a good opportunity to put something live on another cloud platform to get a good comparison, see the pro/cons, etc.

I have to say that working with GCloud isn't that much different. The services it offers are mostly comparable to AWS'.
At least the ones I want to use anyway.

In this article we're going to go through the architectural details of our new webapp.

<a href="/assets/images/articles/blu_architecture_v1.jpg">![](/assets/images/articles/blu_architecture_v1.jpg "Architecture overview")</a>

*(High-level infrastructure diagram)*

# Goals
When I started planning, I had a few basic requirements in mind regarding the operation and maintainability of the system.

* Proper source control
* End-to-end CI/CD pipeline
* Scalability
* Stateless execution, and immutable backend
* Cost efficiency

To be fair, I should have started with a very simple walking skeleton, but I've already had an Angular application and an API ready for deployment, so it was a bit more complicated to put everything in place.

## Source Control
This was a no-brainer, as I love Git. I already have a Github subscription, so I created a new private repo, and that's pretty much it.

## CI/CD
I needed a continuous integration/deployment system which can build containers, runs on linux, has a bunch of 3rdParty integration, and **free**. I have to be honest, I started out with [VSOnline](https://www.visualstudio.com/vso/) but backed-out halfway through as I found it too MS tech focused (geez wonder why :)).

So I fell back to my usual choice: [CircleCI](https://circleci.com/).
I've been using Circle since the early days, and find it just right for my projects. They recently released Circle 2.0 with a whole lot of new features. Their [free-tier](https://circleci.com/pricing/) is amazing, you should be able to do pretty much everything you need for a small-medium sized project.

It nicely integrates with [Github](https://github.com), [Slack](https://slack.com/), [GCloud](https://cloud.google.com/) / [AWS](https://aws.amazon.com/). With a few click you can get everything up and running...*as long as it's a simple hello world*.

## Scalability
As I mentioned before, I decided to go with [Kubernetes](https://kubernetes.io).
Kubernetes is a container orchestrator allowing us to efficiently run, manage and scale containers on virtual or physical machines. It also provides self-healing, and zero-downtime deployments (and much-much more).
I had bad experiences with spinning it up in AWS, but on GCloud it's a breeze.
This, accompanied by Google CDN for static files/images hosting will suit my needs well on the long run.

I chose Cloud Datastore as my DB engine, so I don't have to deal with DB bottleneck, replication, and whatnot. It will nicely scale with my traffic, and that is good enough for now.

## Stateless execution / immutability
I decided **not** to use sessions in my backend implementation as they're are generally considered to be a bad concept. I strongly suggest you to check out [this article](https://brockallen.com/2012/04/07/think-twice-about-using-session-state/) for details.
From the infrastracture viewpoint, it eliminates some complications from the backend. Thanks to this, it becomes very easy to horizontally scale the infrastructure without the hassle of managing a Redis/MemCached/etc cluster.

I'm encoding all necessary information in a JWT token, which is then used for authentication and basic data storage for the frontend (like user details).

## Cost efficiency
With the current setup, I only have to pay for the followings:

* Github account
* Domain name
* 1 VM for my node (I'll scale this up to 3 nodes before launching my site)
* Container registry (peanuts)

Google is nice enough to give you **$300** after signing up for a cloud account, which can be used during the first 12months.

## One click end-to-end deployment process
If you want to make your life easier, you better automate everything. You'll never have to worry about releasing anything ever again.

Let's see how a typical release cycle looks like:

* You push your code to your master branch.
* Via webhook integration, Circle picks up the changes and kicks off the build:
  * Compile the 2 projects (frontend, and backend)
  * Build 2 docker images from the build artifacts
  * Push the new images to the private container registry, tagged with the build number
  * Use kubectl to update the image versions in the cluster
  * Send me a message on Slack with the build details
* The image version update triggers a Kubernetes deployment update, which does a zero-downtime upgrade

The whole process takes **less than 5 minutes**. Even this can be halved by subscribing to Circle's layer caching service, which prevents the build server from re-downloading the build container images (we're talking about 2-3Gb here - *ouch!*).

## Improvement areas
The current setup is far from complete; or production ready. It's definitely a nice start, but it still misses a few key requirements for normal operation.

I'm planning to add the following features during the next few months:
* Log aggregation (Stackdriver / ELK stack)
* Metrics and monitoring/alerting
* Blue/Green deployment
* Infrastructure-as-code (half done)

# Coming up next...
I didn't want to squeeze everything into this blog post, so I'm going to follow this up with another one going through the implementation details.
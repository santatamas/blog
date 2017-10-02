---
layout: post
title: Building Blu - Infrastructure II.
date:   2017-10-02 01:31:00 +0100
categories: Blu Cloud Kubernetes dotnetcore CDN DevOps Infrastructure
---

In the [previous post](/Building-Blu-Infrastructure/) we had a look at the high-level overview of the infrastructure of our shiny new webapp. Now it's time to take a deep-dive into the implementation details. I know you're stoked to see some actual code, so here we go!

# Dockerising the applications
In order to be able to deploy our apps to the Kubernetes cluster, first we have to build Docker container images for them. If you remember, we're dealing with an Angular4 SPA, and a dotnetcore backend.

You might wonder why I chose to use a container instead of using Google CDN to distribute the static html/JS files for the SPA. While the latter might seem to be the straightforward choice in most cases, I have several reasons why I you'd want to go with the other.

* Container makes versioning easy. I can tag my container with the build number, and push it to Kubernetes, which means I can always easily roll back any changes without a fuss.
* Invalidating CDN caches is *ssllloooww*. It's not a big deal, but if and when the house is on fire, it's a pain to push out anything quickly.
* It's a well-known practice to put a proxy in front of [Kestrel](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?tabs=aspnetcore2x), as it's not necessarily ready to be exposed to the public.

These reasons lead me to settle with an Nginx container, which is responsible for serving static content, and proxy all traffic to **/api** to my backend.

Fortunately, configuring Nginx as a proxy is *REALLY* easy.
``` javascript
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
        expires 1h;
        add_header Cache-Control "public";
    }

    location /api {
        proxy_pass http://localhost:8080;
    }
}
```
In this config you can see that Nginx is serving all static content (and index.html by default) on the root, and forwards all calls from **/api** to my backend Api, running on port 8080. The **localhost** url is right in this case, as these 2 containers will sit in the same [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/), so they can communicate through the loopback interface.

Let's wrap this bad boy with our SPA into a container.
*(if you're not familiar with [Dockerfiles](https://docs.docker.com/engine/reference/builder/), now is the right time to start ;))*
```
FROM nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY production /usr/share/nginx/html
```
What happens here is that we take the official Nginx image, copy the config and contents of the production folder - (Angular release artifacts).

So far so good. If you build this and spin it up, you'll be able to navigate to `http://localhost`, and see the app running:
``` shell
docker build -t myproject/blu-spa:latest .
docker run -d -p 80:80 myproject/blu-spa:latest
```

Now onto the backend. This one isn't tricky either.
```
FROM microsoft/aspnetcore:2.0.0
WORKDIR /app
COPY ./publish .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "Blu.Api.dll"]
```

You can run it the same way as the web container:
``` shell
docker build -t myproject/blu-api:latest .
docker run -d -p 8080:8080 myproject/blu-api:latest
```

Now you can push them to your preferred container registry.
I'm using Google Container Registry, which is nicely supported by the CLI tool. If you installed the Google SDK already, all you have to do is:
```
docker tag myproject/blu-api:latest eu.gcr.io/{GOOGLE_PROJECT_ID}/blu-api:latest
docker tag myproject/blu-spa:latest eu.gcr.io/{GOOGLE_PROJECT_ID}/blu-spa:latest
gcloud docker -- push eu.gcr.io/{GOOGLE_PROJECT_ID}/blu-api:latest
gcloud docker -- push eu.gcr.io/{GOOGLE_PROJECT_ID}/blu-spa:latest
```

*Whew! This wasn't too bad right?*

# Setting up a Kubernetes Cluster
First, let's take a closer look to our Kubernetes cluster. As a reminder, here how it looks like:
<a href="/images/articles/infrastructure/kubernetes_cluster.png">![](/images/articles/infrastructure/kubernetes_cluster.png "Kubernetes Cluster")</a>

Before I go into details, there are a few Kubernetes terminology you'll have to be familiar with in order to get what's going on with this setup.

* [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
* [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Services](https://kubernetes.io/docs/concepts/services-networking/service/)

Each Node is a VM running in GCloud, running a single instance of a Pod - containing 2 docker containers with our NginX and Api images.

With one node in place this works just fine. But as soon as we want to scale up, we have to introduce a `Service` to gain a single point of entry to our infrastructure. The following quite summarises this concept perfectly:
> Kubernetes Pods are mortal. They are born and when they die, they are not resurrected.
> A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them -
> sometimes called a micro-service. The set of Pods targeted by a Service is (usually) determined by a Label Selector
> (see below for why you might want a Service without a selector).

Fortunately you can configure everything in yaml files - so without saying more, check out the configurations.

Service:
``` yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kompose.cmd: kompose convert
    kompose.version: 1.2.0 ()
  creationTimestamp: null
  labels:
    io.kompose.service: blu-service
  name: blu-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    io.kompose.service: blu-service
status:
  loadBalancer: {}
```

Deployment:
``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    io.kompose.service: blu-service
  name: blu-pod
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        io.kompose.service: blu-service
    spec:
      containers:
      - image: eu.gcr.io/{GOOGLE_PROJECT_ID}/blu-api:latest
        name: blu-api
        env:
        - name: GCLIUD_PROJECT_ID
          value: {GOOGLE_PROJECT_ID}
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: ./google_creds.json
        - name: JWT_ISSUER
          value: {YOUR_DOMAIN}
        - name: JWT_PRIVATE_KEY_FILE
          value: ./blu-dev.pem
        - name: JWT_PUBLIC_KEY_FILE
          value: ./blu-dev.pub.pem
        resources: {}
      - image: eu.gcr.io/s{GOOGLE_PROJECT_ID}/blu-spa:latest
        name: blu-spa
        resources: {}
      restartPolicy: Always
status: {}
```

We can use [Kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/) to create the deployment and service in our cluster:
```
kubectl create -f blu-deployment.yaml
kubectl create -f blu-service.yaml
```

If you check your cluster now, you should be able to see your deployment up and running.



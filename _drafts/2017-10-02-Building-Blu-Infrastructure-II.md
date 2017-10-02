---
layout: post
title: Building Blu - Infrastructure II.
date:   2017-10-02 01:31:00 +0100
categories: Blu Cloud Kubernetes dotnetcore CDN DevOps Infrastructure
---
# Creating application images

Host container for aspnetcore webApi
```
FROM microsoft/aspnetcore:2.0.0
WORKDIR /app
COPY ./publish .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "Blu.Api.dll"]
```

Host container for Angular SPA + api proxy
```
FROM nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY production /usr/share/nginx/html
```

# Nginx configuration
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

# Setting up a Kubernetes Cluster
Service configuration:
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

Deployment configuration:
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
      - image: eu.gcr.io/skilled-array-168418/blu-api:21
        name: blu-api
        env:
        - name: GCLIUD_PROJECT_ID
          value: skilled-array-168418
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: ./google_creds.json
        - name: JWT_ISSUER
          value: ohmy.watch
        - name: JWT_PRIVATE_KEY_FILE
          value: ./blu-dev.pem
        - name: JWT_PUBLIC_KEY_FILE
          value: ./blu-dev.pub.pem
        resources: {}
      - image: eu.gcr.io/skilled-array-168418/blu-spa:21
        name: blu-spa
        resources: {}
      restartPolicy: Always
status: {}
```

# Creating build image
```
FROM microsoft/aspnetcore-build:2.0-jessie
WORKDIR /source

# Install sudo/make/bzip2
RUN apt-get update && apt-get install -y sudo && sudo apt-get install make && sudo apt-get install bzip2

# Install Google Cloud SDK
ENV CLOUD_SDK_REPO=cloud-sdk-jessie
RUN echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
    sudo apt-get update && sudo apt-get install google-cloud-sdk -y && \
    sudo apt-get install kubectl -y

# Install Angular-CLI
RUN npm install -g @angular/cli

ENTRYPOINT ["bash"]
```

# Configuring Circle

``` yaml
version: 2

jobs:
   build:
     docker:
       - image: tamassanta/blu-circle:latest
     steps:
       - checkout
       - run:
          name: Decode Google Cloud Credentials
          command: echo ${GOOGLE_AUTH} | base64 -i --decode > ${HOME}/gcp-key.json
       - run: gcloud auth activate-service-account --key-file ${HOME}/gcp-key.json
       - run: gcloud --quiet config set project ${GOOGLE_PROJECT_ID}
       - run: make restore
       - run: make build
       - run: make publish
       - run: make publish-spa
       - setup_remote_docker
       - run:
          name: Install Docker client
          command: |
            set -x
            VER="17.03.0-ce"
            curl -L -o /tmp/docker-$VER.tgz https://get.docker.com/builds/Linux/x86_64/docker-$VER.tgz
            tar -xz -C /tmp -f /tmp/docker-$VER.tgz
            mv /tmp/docker/* /usr/bin
       - run: make build-containers
       - run: make push-containers TAG=${CIRCLE_BUILD_NUM}
       - run: gcloud container clusters get-credentials blu-cluster-1 --zone europe-west3-a --project {GOOGLE_PROJECT_ID}
       - run: kubectl set image deployment/blu-pod blu-api=eu.gcr.io/skilled-array-168418/blu-api:${CIRCLE_BUILD_NUM}
       - run: kubectl set image deployment/blu-pod blu-spa=eu.gcr.io/skilled-array-168418/blu-spa:${CIRCLE_BUILD_NUM}
```


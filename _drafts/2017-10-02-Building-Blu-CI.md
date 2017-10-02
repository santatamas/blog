

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








gcloud container clusters get-credentials blu-cluster-1 --zone europe-west3-a --project skilled-array-168418
kubectl proxy

docker tag f5ca3e57b192 eu.gcr.io/skilled-array-168418/blu-api:latest
gcloud docker -- push eu.gcr.io/skilled-array-168418/blu-api:latest

kubectl set image deployment/blu blu=eu.gcr.io/skilled-array-168418/blu-api:10
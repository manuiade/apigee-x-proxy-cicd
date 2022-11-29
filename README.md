# Apigee X Deploy Proxy Continous Deployment


## Prerequirements

- You have a GCP Project with an evaluation Apigee X Organization setup

## Environemnt variables
```
PROJECT_ID=<YOUR_PROJECT_ID>
SA_NAME=cloud-build-apigee
REPO_NAME=apigee-x-proxy-cicd
GCP_REGION=europe-west1
BUCKET_NAME=<PROXY_BUCKET_NAME>
TOPIC_NAME=<TOPIC_NAME>

gcloud config set $PROJECT_ID
```

## Setup bucket and PubSub notification for object finalize events

```
gsutil mb -p $PROJECT_ID -l $GCP_REGION gs://$BUCKET_NAME
gcloud pubsub topics create $TOPIC_NAME

gsutil notification create -t $TOPIC_NAME -f json -e OBJECT_FINALIZE gs://$BUCKET_NAME
```

## Setup Cloud Build

### Create SA and grant access to Apigee X Organization and write logs to cloud build
```
gcloud iam service-accounts create $SA_NAME

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com --role=roles/apigee.admin
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com --role=roles/logging.logWriter

gsutil iam ch serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com:roles/storage.objectViewer gs://$BUCKET_NAME
```

### Create trigger for openapi spec and wsdl spec

```
# Openapi trigger
gcloud alpha builds triggers create pubsub \
  --name=apigee-openapi-trigger \
  --topic=projects/$PROJECT_ID/topics/$TOPIC_NAME \
  --build-config=cloudbuild_openapi.yaml \
  --substitutions _EVENT_TYPE='$(body.message.attributes.eventType)',_BUCKET_ID='$(body.message.attributes.bucketId)',_OBJECT_ID='$(body.message.attributes.objectId)' \
  --subscription-filter='_EVENT_TYPE == "OBJECT_FINALIZE" && _OBJECT_ID.matches("^openapi\\/.*\\.yaml$")' \
  --branch=master \
  --service-account "projects/$PROJECT_ID/serviceAccounts/$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
  --repo=https://github.com/manuel-injenia/apigee-x-proxy-cicd \
  --repo-type GITHUB

# WSDL trigger
gcloud alpha builds triggers create pubsub \
  --name=apigee-wsdl-trigger \
  --topic=projects/$PROJECT_ID/topics/$TOPIC_NAME \
  --build-config=cloudbuild_wsdl.yaml \
  --substitutions _EVENT_TYPE='$(body.message.attributes.eventType)',_BUCKET_ID='$(body.message.attributes.bucketId)',_OBJECT_ID='$(body.message.attributes.objectId)' \
  --subscription-filter='_EVENT_TYPE == "OBJECT_FINALIZE" && _OBJECT_ID.matches("^wsdl\\/.*\\.wsdl$")' \
  --branch=master \
  --service-account "projects/$PROJECT_ID/serviceAccounts/$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
  --repo=https://github.com/manuel-injenia/apigee-x-proxy-cicd \
  --repo-type GITHUB
```

### Test triggers
```
# Openapi
gsutil cp BookService.wsdl gs://${BUCKET_NAME}/wsdl/BookService.wsdl

# WSDL
gsutil cp mocktarget.yaml gs://${BUCKET_NAME}/openapi/mocktarget.yaml
```
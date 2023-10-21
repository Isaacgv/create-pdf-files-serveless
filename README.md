
# Serverless App with Cloud Run that Creates PDF Files
Build a PDF converter web app on Cloud Run that automatically converts files stored in Cloud Storage into PDFs stored in separate folders.


## Architecture

![App](app.png)


1. Extracts the file details from the Pub/Sub notification.
2. Downloads the file from Cloud Storage to the local hard drive. 
3. Converts the downloaded file to PDF.
4. Uploads the PDF file to Cloud Storage. 
5. Deletes the original file from Cloud Storage.


## To build and deploy the REST API 

Start the build process:
```
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```

To deploy the application
```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region Lab Region \
  --no-allow-unauthenticated \
  --max-instances=1
```

Create the environment variable $SERVICE_URL for the app
```
SERVICE_URL=$(gcloud beta run services describe pdf-converter --platform managed --region Lab Region --format="value(status.url)")
```

POST request to the service
```
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

## Trigger the Cloud Run service when a new file is uploaded

Create a bucket in Cloud Storage for the uploaded docs
```
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-upload
```

Create a bucket for the processed PDFs
```
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-processed
```

To tell Cloud Storage to send a Pub/Sub notification whenever a new file has finished uploading to the docs bucket
```
gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload

```

Give the new service account permission to invoke the PDF converter service
```
gcloud beta run services add-iam-policy-binding pdf-converter --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region Lab Region
```

Enable the project to create Cloud Pub/Sub authentication tokens
```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```

Create a Pub/Sub subscription
```
gcloud beta pubsub subscriptions create pdf-conv-sub --topic new-doc --push-endpoint=$SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

## Testing
Create a Pub/Sub subscription
```
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

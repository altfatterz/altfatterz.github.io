---
layout: post title: Cloud Functions Demo tags: [gcp]
---

```bash
$ Enable APIs
$ gcloud services enable cloudfunctions.googleapis.com
$ gcloud services enable cloudbuild.googleapis.com
```

List regions available to Google Cloud Functions.

```bash
$ gcloud functions regions list
```

List types of events that can be a trigger for a Google Cloud Function.

```bash
$ gcloud functions event-types list
```

List Google Cloud Functions.

```bash
$ gcloud functions list
```

Display details of a Google Cloud Function.

```bash
$ gcloud functions describe function-1 --region europe-west6
```

Create or update a Google Cloud Function.

```bash
$ gcloud functions deploy hello-world-java \
--entry-point com.example.HelloWorldJavaCloudFunction \
--runtime java11 \
--memory 512MB --trigger-http --allow-unauthenticated \
--region europe-west6 
```
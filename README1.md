# Google Cloud and AWS Sync Setup: Client and DataOrb Configurations

## Overview

This document provides step-by-step instructions for configuring Google Cloud and AWS synchronization, split into two main sections: Client Account Setup and DataOrb Account Setup.

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Environment Variables](#environment-variables)
4. [Client Account Configuration](#client-account-configuration)
5. [DataOrb Account Configuration](#dataorb-account-configuration)
6. [Deployment of `gcs_to_s3_function`](#deployment-of-gcs_to_s3_function)
7. [Validation and Testing](#validation-and-testing)

---

## 1. Introduction

This guide explains how to set up Google Cloud and AWS synchronization, with dedicated steps for both the **Client Account** and the **DataOrb Account**.

## 2. Prerequisites

- Google Cloud SDK installed
- AWS Access and Secret keys
- Billing account configured in Google Cloud
- Required permissions to create projects and enable APIs in Google Cloud
- Client Project Number for Client Side

## 3. Environment Variables

Define environment variables for use throughout the setup:

```bash
# Define Variables
BILLING_ACCOUNT_ID=01690A-D59341-4956A3

CLIENT_PROJECT_ID=tww-ccs-analytics-prod
CLIENT_PROFILE=tww-ccs
CLIENT_REGION=us-east1
CLIENT_BUCKET=tww-ccs-analytics-prod--eci

DATAORB_PROJECT_ID=dataorb-gcs-sync07
DATAORB_PROFILE=dataorb-gcs-sync
DATAORB_REGION=us-central1

AWS_ACCESS_KEY_ID=ASDASDASFSAFSDDAS
AWS_SECRET_ACCESS_KEY=SFSDFSDFSDFAEWQ23423422
AWS_S3_BUCKET_NAME=dataorb-gcp-sync01
AWS_REGION_NAME=us-east-1
```

---

## 4. Client Account Configuration

The Client Account setup involves creating a new Google Cloud project, configuring necessary permissions, and setting up the GCS bucket.

### 4.1 Create and Configure Client Project

1. **Create Client Project:**
   ```bash
   gcloud projects create $CLIENT_PROJECT_ID
   ```
2. **Set Client Configuration:**

   ```bash
   gcloud config configurations create $CLIENT_PROFILE
   gcloud config configurations activate $CLIENT_PROFILE
   gcloud config set project $CLIENT_PROJECT_ID
   gcloud config set account kalpesh.p@dataorb.ai
   gcloud config set run/region $CLIENT_REGION
   gcloud config set run/platform managed
   gcloud config set eventarc/location $CLIENT_REGION
   ```

### 4.2 Enable Required Services on Client Account

```bash
export CLOUDSDK_ACTIVE_CONFIG_NAME=$CLIENT_PROFILE
gcloud services enable \
    storage.googleapis.com \
    pubsub.googleapis.com \
    cloudfunctions.googleapis.com
```

### 4.3 Create Google Cloud Storage Bucket

```bash
gcloud storage buckets create gs://$CLIENT_BUCKET/ --location=$CLIENT_REGION
```

---

## 5. DataOrb Account Configuration

This section covers the DataOrb account setup, which includes creating the project, configuring IAM roles, and enabling services needed for the GCS-to-S3 sync function.

### 5.1 Create and Configure DataOrb Project

1. **Create DataOrb Project:**

   ```bash
   gcloud projects create $DATAORB_PROJECT_ID
   ```

2. **Set DataOrb Configuration:**

   ```bash
   gcloud config configurations create $DATAORB_PROFILE
   gcloud config configurations activate $DATAORB_PROFILE
   gcloud config set project $DATAORB_PROJECT_ID
   gcloud config set account kalpesh.p@dataorb.ai
   gcloud config set run/region $DATAORB_REGION
   gcloud config set run/platform managed
   gcloud config set eventarc/location $DATAORB_REGION
   ```

3. **Link Billing Account:**
   ```bash
   gcloud beta billing projects link $DATAORB_PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
   ```

### 5.2 Enable Required Services on DataOrb Account

```bash
export CLOUDSDK_ACTIVE_CONFIG_NAME=$DATAORB_PROFILE
gcloud services enable \
    storage.googleapis.com \
    pubsub.googleapis.com \
    cloudfunctions.googleapis.com \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    eventarc.googleapis.com
```

### 5.3 Set Up IAM for DataOrb Access

1. **Create Service Account for DataOrb Bucket Reader:**

   ```bash
   gcloud iam service-accounts create dataorb-bucket-reader \
       --description="Service account for reading objects from $CLIENT_BUCKET" \
       --display-name="DataOrb Client Reader"
   ```

2. **Generate Key for Service Account:**

   ```bash
   gcloud iam service-accounts keys create ./sync_gcs_to_s3/dataorb-bucket-reader.json \
       --iam-account=dataorb-bucket-reader@$DATAORB_PROJECT_ID.iam.gserviceaccount.com
   ```

3. **Create Pub/Sub Topic for Notifications:**

   ```bash
   gcloud pubsub topics create gcs-to-s3-topic --project=$DATAORB_PROJECT_ID
   ```

4. **Give Access to Client Prject for Pub/Sub:**
   ```bash
   gcloud pubsub topics add-iam-policy-binding gcs-to-s3-topic \
    --project=$DATAORB_PROJECT_ID \
    --member=serviceAccount:service-$CLIENT_PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com \
    --role=roles/pubsub.publisher
   ```

---

## 6. Deployment of `gcs_to_s3_function`

### 6.1 Retrieve Client Project Number in Client Account

```bash
CLIENT_PROJECT_NUMBER=$(gcloud projects describe $CLIENT_PROJECT_ID --format='value(projectNumber)')
```

### 6.2 Configure Pub/Sub and IAM Permissions in DataOrb Account

1. **Add IAM Policy Binding for Pub/Sub:**

   ```bash
   gcloud pubsub topics add-iam-policy-binding gcs-to-s3-topic \
       --project=$DATAORB_PROJECT_ID \
       --member=serviceAccount:service-$CLIENT_PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com \
       --role=roles/pubsub.publisher
   ```

### 6.3 Object Viewer Access and Bucket Notification in Client Account

1. **Grant Object Viewer Access to DataOrb Service Account:**

   ```bash
   gsutil iam ch \
       serviceAccount:dataorb-bucket-reader@$DATAORB_PROJECT_ID.iam.gserviceaccount.com:objectViewer \
       gs://$CLIENT_BUCKET
   ```

2. **Create GCS Bucket Notification:**
   ```bash
   gcloud alpha storage buckets notifications create \
       --topic=projects/$DATAORB_PROJECT_ID/topics/gcs-to-s3-topic \
       --payload-format=json \
       --event-types=OBJECT_FINALIZE \
       --skip-topic-setup \
       gs://$CLIENT_BUCKET
   ```

### 6.4 Deploy the Cloud Function in DataOrb Account

```bash
gcloud functions deploy gcs_to_s3_function --gen2 \
    --runtime python310 \
    --trigger-topic gcs-to-s3-topic \
    --project=$DATAORB_PROJECT_ID \
    --entry-point=sync_gcs_to_s3 \
    --region=$DATAORB_REGION \
    --timeout=120s \
    --set-env-vars AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID,AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY,AWS_S3_BUCKET_NAME=$AWS_S3_BUCKET_NAME,GCP_CREDENTIALS_JSON_PATH=dataorb-bucket-reader.json \
    --source ./sync_gcs_to_s3
```

---

## 7. Validation and Testing

- Verify that the function successfully syncs data from the Google Cloud bucket to the AWS S3 bucket.
- Test by uploading files to the GCS bucket and ensuring they appear in the AWS S3 bucket.

---

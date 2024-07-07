# Cloud Build Slack Notifier

This notifier uses [Slack Webhooks](https://api.slack.com/messaging/webhooks) to send notifications to your Slack workspace channel.

This notifier runs as a container via Google Cloud Run For more detail about setting for this notifier, see [Configuring Slack notifications](https://cloud.google.com/cloud-build/docs/configuring-notifications/configure-slack).
![Slack Notifier](CloudBuild-Slack.svg)
### Configuring Slack notifications

    You need correct right in Slack Workspace to create New Application
    Go to link https://api.slack.com/apps and push bottom Create New App 
![Create New App](README/1.png)

    Select From scratch if we not have manifest (we really don't have it :))
![Create an app](README/2.png)

    In form App Name enter name of Application, after that select your workspace
![App Name](README/3.png)

    After creating App found topic Building Apps for Slack and tap Incoming Webhooks
![Incoming Webhooks](README/4.png)

    On page Incoming Webhooks tap on bottom "Add New Webhook to Workspace"
![Incoming Webhooks](README/5.png)

    On page with permission on Workspace choose in drop down menu channel with notification and tap Allow
![choose channel](README/6.png)

    Congratulation we created Webhook on defined Channel in Workspace. Don't forget copy and store Webhook URL 
![choose channel](README/7.png)

    Don't forget copy and store Webhook URL as Secret  in GCP Secret Manager

```commandline
echo -n "https://hooks.slack.com/services/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" | gcloud secrets create WEBHOOK_KEY --data-file=-
```
### Configuring Cloud Build Notifiers

Let's go to GCP side.
For run notification you need to store generated before Webhook URL in Secret Manager, call it `WEBHOOK_KEY`

To get notification in slack we need rollout Cloud Run with Image

 official guide https://github.com/GoogleCloudPlatform/cloud-build-notifiers
 
But I took the time to find exactly the information I needed to install only Cloud Build Notifiers for notification in Slack 

If shortly - for run Cloud Run we need to create two files which need to place in GCS bucket
Create the bucket 
gsutil rb -l europe-west1 gs://vetapp2date-af5f0-gcp_cloudbuild_slack_webhook

### File `slack.json` 

slack.json - leave as is and put it in GCS bucket
Copy file from repository to bucket
```commandline
gsutil cp slack.json gs://vetapp2date-af5f0-gcp_cloudbuild_slack_webhook
```

### File `slack.yaml` 
slack.yaml - make some changes. Modify next field in file

To Inform about Success and Fail builds in `spec.notification.filter` change value on
```commandline 
filter: build.status == Build.Status.SUCCESS || build.status == Build.Status.FAILURE
```

In `spec.notification.template.uri` change value on
```commandline 
uri: gs://vetapp2date-af5f0-gcp_cloudbuild_slack_webhook/slack.json
```

In `spec.secrets.value` change value on
```commandline 
value: projects/vetapp2date-af5f0/secrets/WEBHOOK_KEY/versions/latest
```

Copy file slack.yaml from repository to bucket
```commandline
gsutil cp slack.yaml gs://vetapp2date-af5f0-gcp_cloudbuild_slack_webhook
```

Now all is ready to install in Cloud Run our Cloud Build Notifiers for Slack
- Trigger for Cloud Build MUST Create on [previous step](https://github.com/dnk80/CloudBuild/tree/main/CloudBuild-GitHub-integration) in other case you got error and Cloud Run not started.
- For Service Account MUST grant role Secret Manager Secret Accessor (--role=roles/run.invoker)


## Install Slack Notifier
```commandline
gcloud run deploy service-name \
--region=europe-west1 \
--image=us-east1-docker.pkg.dev/gcb-release/cloud-build-notifiers/slack:slack-1.15.0 \
--no-allow-unauthenticated \
--update-env-vars=CONFIG_PATH=gs://<GCS_NAME>/slack.yaml,PROJECT_ID=<PROJECT_NAME> \
--service-account=<SA_NAME>@<PROJECT_NAME>.iam.gserviceaccount.com
```

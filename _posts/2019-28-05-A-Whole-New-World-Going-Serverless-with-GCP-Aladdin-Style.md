---
title: 'A Whole New World: Going Serverless with GCP (Aladdin-style)'
date: 2019-05-28
permalink: /blog/A-Whole-New-World-Going-Serverless-with-GCP-Aladdin-Style/
tags:
  - python
  - GCP
  - serverless
  - data
  - retrieval
  - spotify
  - api
  - cloud  
  - bigquery
---  
<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/aladdin-cover.jpg?raw=true">
</p>

<br/>
<br/>

### Once upon a time, a young street urchin from a kingdom far, far away was managing virtual machines and cron jobs for a living but soon his life will turn upside down when he discovers serverless.

Having a setup where a simple script is running once a day/week/month on a server is not ideal. Sure, this could be a small, free-tier virtual machine running in a public cloud, so who cares?

In this article I will give a quick how-to for replacing a simple data retrieval pipeline (as the one described [in this article](https://tgel0.github.io/blog/spotify-data-project-part-1-from-data-retrieval-to-first-insights/)) with a completely serverless solution using the Google Cloud Platform.

Note:  there might be a reference or two from Disney's Aladdin movie which I recently watched!

Let's imagine ourselves in the ancient kingdom of [Agrabah](https://onceuponatime.fandom.com/wiki/Agrabah). The protagonist of our story was looking for a new solution for his data pipeline when a giant, blue ghost came out of a magic lamp.

<br/>

![](https://media.giphy.com/media/4K3l1T9MZwtoivFqtx/giphy.gif)

<br/>

## 🧞 Three Wishes

 

The Genie granted three wishes:

- I wish to never have to worry about my infrastructure ever again
- I wish to only pay for the compute time my code is using to run
- and finally, I wish automatic scaling with high availability

<br/>

## ☁️ Answer is serverless

Back in the old days, this would probably seem impossible even for Big G.

But today all the major cloud providers have some sort of serverless option: AWS Lambda, MS Azure Functions, GCP Cloud Functions.

In this project, I implemented the following architecture to get a weekly dataset of audio data from the Spotify API:

<p align="center">
<img align="center" src="https://raw.githubusercontent.com/tgel0/tgel0.github.io/master/images/GCP_diagram.png">
</p>

In more detail this means:

☁️ the first [Cloud Function](https://cloud.google.com/functions/) is retrieving the data from the Spotify API and storing it as CSV

☁️  [Cloud Storage](https://cloud.google.com/storage/) is just for storing the CSVs

☁️ the second CF will load the data from the CSVs into BigQuery

☁️  [BigQuery](https://cloud.google.com/bigquery/) is our serverless data warehouse holding all the data

☁️  last but not least, [Cloud Scheduler](https://cloud.google.com/scheduler/) (which is not shown in the diagram above) will trigger the first CF once a week

<br/>

## ☝️Command Line vs. Web Console

Interacting with GCP services can be done via the CLI or web console (REST API being another option). Personally, I find the web console very user-friendly and can absolutely recommend it. In fact, I did this project mostly using the web frontend.

<br/>

### ☁️ Cloud Functions

Two Cloud Functions were needed here: one for retrieving the data from Spotify and the other one for loading the data into BigQuery.

Besides the function itself there is one major difference between them: the trigger.

The first one is going to be triggered by an HTTP request (like opening an URL with a web browser). The second one by a Cloud Storage event (whenever a new CSV is stored in the Cloud Storage bucket).

<br/>

🐍 Gotcha alert: my Cloud Function was crashing until I added a 'greater-than' sign in my `requirements.txt` file. 

<br/>

So I basically changed this:

<br/>

    pandas==0.22.0
    spotipy==2.4.4
    google-cloud-storage==1.15.0
    
<br/>

Into this:

    pandas>=0.22.0
    spotipy>=2.4.4
    google-cloud-storage>=1.15.0

<br/>

The second Cloud Function is going to take the CSV's from the Google Storage bucket and load them into BigQuery.

<br/>

### ☁️  BigQuery

Setting up a dataset and table in BigQuery is fairly easy with the the web console. Just make sure to not miss any columns or data types.

<br/>

<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/spotify_data_bigq.PNG?raw=true">
</p>

<br/>

### ☁️  Cloud Scheduler

We could now trigger the whole data pipeline manually by opening the URL endpoint of the first Cloud Function in a web browser. But that's not how we do things *here in the kingdom of Agrabah* 🧞.

With the help of [Cloud Scheduler](https://cloud.google.com/scheduler/) we will automatically trigger the URL endpoint at a specified schedule.

Below is the schedule for "every Monday at 12:30 AM". Target/method needs to be HTTP/POST.

<br/>

    30 00 * * 1

<br/>

🐍 Picking a schedule using the unix-cron format is another gotcha. [This website](https://crontab.guru/) can help.

<br/>

After everything was set up we can just fly away on our magic carpet and let the automation do it's thing.

<br/>

![](https://media.giphy.com/media/WUu9EGdSEImJy/giphy.gif)

<br/>
<br/>

## T H E        E N D

<br/>

## References

<br/>

🔗 [Moving your cron job to the cloud with Google Cloud Functions](https://dev.to/di/moving-your-cron-job-to-the-cloud-with-google-cloud-functions-1ecp?__s=4oo7uhefd3rtrvzokcnh) by Dustin Ingram

🔗 [Automated insert of CSV data into Bigquery via GCS bucket + Python](https://rickt.org/2018/10/22/poc-automated-insert-of-csv-data-into-bigquery-via-gcs-bucket-python/) by Rick Tait

🔗 My [github repo](https://github.com/tgel0/spotify-data) for this project (includes all the notebooks, scripts and more)

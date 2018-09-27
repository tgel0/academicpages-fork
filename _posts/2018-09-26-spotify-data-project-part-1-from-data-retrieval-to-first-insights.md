---
title: 'Spotify Data Project Part 1 - from Data Retrieval to First Insights'
date: 2018-09-03
permalink: /blog/most-popular-tracks-2018-spotify/
tags:
  - python
  - data
  - collection
  - spotify
  - api
  - tracks  
  - popular
  - summer
---  
<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/spotipython.png?raw=true">
</p>

<br/>
<br/>


2018 has been a great year so far for us data nerds. There are so many great options to get interesting data, such as the biggest data science community Kaggle, with more than 10.000 published datasets from all kinds of industries. Or Google (who also happens to own Kaggle) with the newly introduced [dataset search tool](https://toolbox.google.com/datasetsearch), makes finding datasets for practicing your data muscles as easy as installing Pandas (or any other data science library for that matter). 

And if you are willing to get your hands dirty for some nice data you can always go for the APIs. Tech companies such as Twitter, Slack and Google (again!) provide APIs so that others can build applications on top of those. There is nothing stopping you from using them for extracting data and doing some data analysis with it.

In this series of articles, I'm going to describe how I utilized the [Spotify Web API](https://developer.spotify.com/documentation/web-api/) to retrieve data automatically (covered in this article, mostly) as well as how to use data science tools such as Python, SQL and Bash to gain insights from the data (to be continued in the next articles).

**Note:** if you are only interested in the coding stuff feel free to skip to the end of this article where you will find all the notebooks, tools and references used for this project.

## Data Retrieval

After creating an account at [developer.spotify.com](https://developer.spotify.com/) you are already good to go. In fact, you can directly go to the [Spotify Web API Console](https://developer.spotify.com/console/) and start exploring the different API endpoints with an easy-to-use interface without the need for any Jupyter notebooks at all! 

Essentially, APIs work on a request/response basis. You ask the API about tracks from the golden years of 2012 and the API responds with some of the best songs of our generation such as [this gem](https://www.youtube.com/watch?v=APUxJ8f7PXY). However, since the communication with the API is done in a machine-readable format such as JSON, you will want to use a programming language to do this. That is, if you want to make use of the millions of rows of music data available in the Spotify catalogue.

<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/SpotifyWebAPI.png?raw=true">
</p>

Since I'm using Python, there is a very nice library called [Spotipy](https://spotipy.readthedocs.io/) which makes the process of accessing the Spotify API a lot easier.

Once you installed Spotipy, the below code is sufficient to get it up and running (use your own cid and secret from your Spotify developer account):

```
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials

cid ="xx" 
secret = "xx"

client_credentials_manager = SpotifyClientCredentials(client_id=cid, client_secret=secret)
sp = spotipy.Spotify(client_credentials_manager=client_credentials_manager)
```
Now you can start with the actual data retrieval. By looking at the different endpoints of the Spotify API you might ask yourself: what data should I get?

<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/Spotifyendpoints.PNG?raw=true">
</p>

This is where you need to start digging and finding out what may be interesting for you. In this example, I used only 2 out of the 50+ endpoints availabe. You might find some other endpoints also useful for your analysis.

I knew that I wanted to work with track data. That's why I used the [search endpoint](https://developer.spotify.com/documentation/web-api/reference/search/search/) (not the [get several tracks endpoint](https://developer.spotify.com/documentation/web-api/reference/tracks/get-several-tracks/) since that one requires track IDs to be provided which I didn't have).

The search endpoint is fairly easy to use as there are only 2 required parameters: q and type. The first one can be any keyword such as 'roadhouse blues' or '2018'. The type can be album , artist, playlist or track.

There are also a couple of limitations with this endpoint:

- limit: a maximum of 50 results can be returned per query
- offset: this is the index of the first result to return, so if you want to get the results with the index 50-100 you will need to set the offset to 50 etc.

Also the maximum offset is limited to 10.000 which means that you can't get more results than that for a single search. **Fun fact:** when I first did the data retrieval (end of April 2018) the maximum offset was 100.000. Hopefully, I was not the reason why they decided to cut it down :)

In order to work around the limitations I used a nested for loop where I increased the offset by 50 in the outer loop until the maxium offset was reached. The inner for loop did the actual querying while appending the returned results to lists.

```
for i in range(0,10000,50):
    track_results = sp.search(q='year:2018', type='track', limit=50,offset=i)
    for i, t in enumerate(track_results['tracks']['items']):
        artist_name.append(t['artists'][0]['name'])
        track_name.append(t['name'])
        track_id.append(t['id'])
        popularity.append(t['popularity'])
```

After I loaded the data into a pandas dataframe, I got the following:

```
	artist_name	popularity	track_id	track_name
  
0	Drake	        100	2G7V7zsVDxg1yRsu7Ew9RJ	In My Feelings
1	XXXTENTACION	97	3ee8Jmje8o58CHK66QrVC2	SAD!
2	Tyga	        96	5IaHrVsrferBYDm0bDyABy	Taste (feat. Offset)
3	Cardi B	        97	58q2HKrzhC3ozto2nDdN4z	I Like It
4	XXXTENTACION	95	0JP9xo3adEtGSdUEISiszL	Moonlight

```

This is already enough to work with, as I will show in the second part of this article when I will dig deeper into the 'popularity' feature provided by Spotify. Since I want to do some more advanced analysis as well (hint: machine learning) I also included data provided by the [get several audio features endpoint](https://developer.spotify.com/documentation/web-api/reference/tracks/get-several-audio-features/).

This endpoint only requires the track IDs to be provided. That wasn't a problem since I had 10.000 track IDs from my previous query.

The limitation here was that a maximum of 100 track IDs can be submitted per query.

Again, I used a nested for loop. This time the outer loop was pulling track IDs in batches of size 100 and the inner for loop was doing the query and appending the results to the rows list.

Additionaly, I had to implement a check when a track ID didn't return any audio features (i.e. None was returned) as this was causing issues:

```
for i in range(0,len(df_tracks['track_id']),batchsize):
    batch = df_tracks['track_id'][i:i+batchsize]
    feature_results = sp.audio_features(batch)
    for i, t in enumerate(feature_results):
        if t == None:
            None_counter = None_counter + 1
        else:
            rows.append(t)
```

In return, I got a bunch of audio features for my tracks such as danceability, loudness, acousticness and so on. Those features however will be part of the next article in this series.

Before doing some data analysis with the popularity feature, I want to show you how I set up a cron job on a Linux terminal (command line/ Bash) to do the data retrieval automatically while I'm sleeping!

It's super simple: you just need your Python file (if you are using Jupyter notebooks you can download it as a .py file) and then you create a Bash script (.sh file) which will run the file in Python:

```
#!/usr/bin/env bash

python3 /home/user/SpotifyDataScript.py
```

Finally, you open your crontab editor and set up a cron job at the desired time and day like this:

```
30     00     *     *     4         /home/user/Spotify.sh
```

And voilà! You don't need to worry anymore, the data will now be retrieved automatically. As you can see from the screenshot below taken from the Spotify for developers dashboard, the data is now being pulled automatically from the API on a weekly basis.

<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/Spotifyapicalls.PNG?raw=true">
</p>

If you want to find out more about Bash scripts and cron jobs I recommend [this series of articles](https://data36.com/data-coding-101-introduction-bash/) from data36 as well as [this quick reference guide](http://www.adminschoice.com/crontab-quick-reference).

**Note:** although this has been working for me quite well I would recommend to be careful when setting up automations for API calls. Try to minimize the impact on the provider side by using longer intervals for triggering the data such as weekly or even monthly.


## Data Exploration: the Popularity Feature

As promised, I will now use the retrieved data to get some insights from it. In this article, I'm going to focus on the popularity feature which, according to the [Official Spotify documentation](https://developer.spotify.com/documentation/web-api/reference/search/search/) means:

>"The popularity of the track. The value will be between 0, for least popular, and 100 for most popular. The popularity of a track is a value between 0 and 100, with 100 being the most popular. Popularity is based mainly on the total number of playbacks. Duplicate tracks, such as both in a single and in an album, are popularity rated differently. Note: This value is not updated in real-time and may therefore lag behind in actual popularity."

Although Spotify isn't being too precise about this feature this should still be enough for a data analysis. Ideally there would be another feature such as the number of streams per track, however, Spotify doesn't share this information via their API at the moment.

An important thing to know about the popularity feature has also been explained in [this SO post from 2013](https://stackoverflow.com/a/19849105/8848786) from a user named Sascha:

>"Spotify's popularity is indeed based on the number of streams, but instead of total plays, it's based on a short timeframe. This is why you'll often see a more popular number on #1 on an artist profile that has less plays than the #2.
>
>We have used the API number as a percentage in our Stream Popularity, giving people insight in the current popularity of their track. So keep in mind this is a number the can increase, but just as easily decrease."

So basically, it represents the popularity of a track *at a given moment*. This explains also why it is changing from time to time as I could notice from my weekly data retrievals.

Since I started collecting data regularly in August 2018, I decided to include the following 3 data retrievals for my analysis:

- 07th August 2018
- 30th August 2018
- 20th September 2018

Thanks to the supposed delay in the calculation of the popularity as mentioned by Spotify, this should cover the months of July, August and September pretty well so that I can market my analysis as the 'most popular tracks of the **summer 2018'**.

For this, I loaded the 3 CSV files into Pandas dataframes and merged them into one single dataframe. By using an outer join as the merging method, the individual popularities were preserved:

```
# loop over dataframes and merge into one dataframe
# outer join in order to keep the popularity column from each file
for df_, files in zip(df_from_each_file, all_files): # all_files are here to provide the column suffix (0920,0830 etc)
    df = df.merge(df_, how='outer', on=merge_columns, suffixes=('',str(files)[-8:-4])
```

This was already the trickiest part of the whole data preparation phase for this analysis. After that, I calculated an overall popularity score as well as a mean popularity score.

The new dataframe sorted by the overall popularity score looks like this:

```
	artist_name	track_name	popularity	popularity_mean
3	Drake	        In My Feelings	300.0	        100.000000
4	XXXTENTACION	SAD!	        288.0	        96.000000
12	Cardi B	        I Like It	288.0	        96.000000
6	Tyga	        Taste (feat. Offset)	287.0	95.666667
10	Post Malone	Better Now	287.0	        95.666667
```

Semms like nothing comes close to Drake's 'In My Feelings' when it comes to summer hits 2018.

Grouping the dataframe by artist reveals some more interesting insights such as the number of tracks per artist in the top 100:

```
	track_name
artist_name	
Drake	        5
XXXTENTACION	5
Travis Scott	5
Post Malone	5
Juice WRLD	3
```

Every good data analysis should also have some visualization. In this example, I visualized the individual popularity scores during the measurement period for the top 10 tracks. This is part of the visualization (check out the notebook file at the end of the article for a full view):

<p align="center">
<img align="center" src="https://github.com/tgel0/tgel0.github.io/blob/master/images/Spotifyviz.PNG?raw=true">
</p>

And that's it for today! A decent amount of data was retrieved and analyzed to find out about the most popular tracks and artists of the summer 2018. 

In the next article, I'm going to dig deeper into the audio features data and try out different clustering algorithms. If that sounds interesting to you, make sure to follow me on Twitter ([@tgel0](https://twitter.com/tgel0)) as I will post the news there first. That's also the best way to reach out to me, if you feel like saying hi or something (I would love that).

As promised, here are the links:

## Notebooks, Tools and References

- Data retrieval notebook on [github](https://github.com/tgel0/spotify-data/blob/master/notebooks/SpotifyDataRetrieval.ipynb) or as [nbviewer render](http://nbviewer.jupyter.org/github/tgel0/spotify-data/blob/master/notebooks/SpotifyDataRetrieval.ipynb)
- Data exploration notebook on [github](https://github.com/tgel0/spotify-data/blob/master/notebooks/SpotifyDataExploPopularity.ipynb) or as [nbviewer render](http://nbviewer.jupyter.org/github/tgel0/spotify-data/blob/master/notebooks/SpotifyDataExploPopularity.ipynb)
- big thanks to Spotify for their [Web API](https://developer.spotify.com/documentation/web-api/) as well as to the creator of [Spotipy](https://spotipy.readthedocs.io/) for making my life easier
- last but not least, this wouldn't be possible without the Python libraries [Pandas](http://pandas.pydata.org/), [Numpy](http://www.numpy.org/), [Matplotlib](https://matplotlib.org/) and the amazing people building and maintaining them
- and if you feel like listening to music instead of just reading about it, I have created a [Spotify playlist](https://open.spotify.com/user/gelotomi/playlist/2bhPeVZnd8llTJt24LjMnE?si=D5oQX3BORIGgB-VsBdcZyg) based on the most popular tracks from this analysis

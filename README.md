# Python Reply Bot

This script replies to all tweets scraped from the search query. It's uses Twitter API v1.1. 
It s fully customizable, so you can change reply text, search keywords and time delays! 

You can set up all configuration if a config file. Here is an example of config. You have to change it to fit your requirements. 
Important: for testing you have to decrease all delays for not waiting for a long.

```sh
[settings]
search_query: #spareroom spareroom.co.uk
# Leave empty for all languages
tweet_language:
reply_text: You can advertise your room for free on www.roompik.com!Find flatmates more easily there than spareroom;You can find like-minded flatmates more easily there than spareroom at  www.roompik.com! It s free!
min_wait_time = 60
max_wait_time = 90
search_delay = 3600
search_depth = 20
# blacklisted users
userBlacklist =

# Create your app on http://apps.twitter.com
[twitter]
consumer_key: 
consumer_secret: 
access_token: 
access_token_secret: 
```

You shouldn't edit any source code. First of all, to start working with the script you should enter all credentials from your twitter app. You need:
consumer_key
consumer_secret
access_token
access_token_secret
Unfortunately, you provided me only with Consumer Key. Therefore, you need to generate all this data. Here is detailed instruction how to do it. If you will have any problems with getting it, please feel free to inform me and i will help you with that.
 
# How it works
The first time the script runs it looks for file with the history of  last the tweet’s id that was replied to avoid replying to the same tweet again. This id will be a startpoint for future search requests. If such file doesn’t exist, the script will consume value of  search_depth  the variable to execute search query. 
For example, if you firts time run script with the  search_depth = 20, it means that the script will reply to last 20 tweets from search result and then stops on search_delay  seconds. After that, it will make search query again, but in this time it will search only that tweets that was published after last tweet in previous iteration. Is there are no new tweets, it will pause again on search_delay  seconds.

The search is executed using search_query, search_depth , tweet_language values. Then, it s filter results by userBlacklist  and delete retweet posts to not to invade other people's conversations.

Next, it scapes tweets data to the console to make you see which tweets are repling now. Also, it s uses randomly taken reply text from reply_text to make it for reliable and avoid bannin.

All this data are saved to the log file(twitter_bot_monitor.log) and if any errors will occur, you can consider it. Here is an example of log:
>2016-07-07 14:14:17,411 | bot_monitor | INFO | Script is running! Please, wait.
>2016-07-07 14:14:17,411 | bot_monitor | WARNING | No previously replied tweets found.Making a search request go get 3 tweets!

>2016-07-07 14:14:21,657 | bot_monitor | INFO | LOADING. Proccesed 1 of the 3 tweets
>2016-07-07 14:14:21,658 | bot_monitor | INFO | REPLIED TO THE TWEET
>2016-07-07 14:14:21,658 | bot_monitor | INFO | (2016-07-07 01:57:02) nomoreMTpots:  #spareroom can certainly be used to grow food. Great #entrepreneurial &amp;  #selfsufficiency effort! #sidehustle https://t.co/ca9ybJSkKy 750871213719719936

>2016-07-07 14:14:21,663 | bot_monitor | INFO | Choosing time between 1 and 1 - waiting 1 seconds before action

>2016-07-07 14:14:22,663 | bot_monitor | INFO | LOADING. Proccesed 2 of the 3 tweets
>2016-07-07 14:14:22,663 | bot_monitor | INFO | REPLIED TO THE TWEET
>2016-07-07 14:14:22,664 | bot_monitor | INFO | (2016-07-07 07:38:57) CaitlinEHam: #spareroom #flat #islington #london https://t.co/0KUivZPm00 750957261590241280

>2016-07-07 14:14:22,664 | bot_monitor | INFO | Choosing time between 1 and 1 - waiting 1 seconds before action

>2016-07-07 14:14:23,664 | bot_monitor | INFO | LOADING. Proccesed 3 of the 3 tweets
>2016-07-07 14:14:23,664 | bot_monitor | INFO | REPLIED TO THE TWEET
>2016-07-07 14:14:23,665 | bot_monitor | INFO | (2016-07-07 10:55:18) 4rentuk: £350pcm, Edgbaston (B17): "Homely Share Near The QE Hospital": Beautiful four bed share in Harborne. The loca... https://t.co/33LudAdwFg 751006673032978432

>2016-07-07 14:14:23,667 | bot_monitor | INFO | Choosing time between 1 and 1 - waiting 1 seconds before action

>2016-07-07 14:14:27,677 | bot_monitor | INFO | Finished. 3 Tweets retweeted, 0 errors occured.
>2016-07-07 14:14:27,677 | bot_monitor | INFO | Script stops for 1 hour

>2016-07-07 14:14:29,678 | bot_monitor | INFO | Script is running! Please, wait.
>2016-07-07 14:14:29,678 | bot_monitor | INFO | Loaded savepoint from reply history. Starting search from tweet id 751010337659482113


In additional, the scripts logs all replied tweet's id to file to not reply it again in the future. 
And after each iteration, it s writes values of last replied id to the file and uses this data in the next search query. 

# How to start 
Download script data and extract to any folder.
Install Python 2.7 and pip.
Install all dependencies by doing next:
``` pip install -r /YOUR/TO/requirements.txt ```
Run the script python 
```$ python Reply_Bot.py```




#!/usr/bin/env python
# -*- coding: utf-8 -*-
import os, tweepy, inspect, hashlib, configparser
from logging.handlers import RotatingFileHandler
import logging
import random
import time
import sys

log = logging.getLogger('bot_monitor')
log.setLevel(logging.DEBUG)
formatter = logging.Formatter(
    '%(asctime)s | %(name)s | %(levelname)s | %(message)s'
)
handler_stdout = logging.StreamHandler(sys.stdout)
handler_stdout.setLevel(logging.DEBUG)
handler_stdout.setFormatter(formatter)
log.addHandler(handler_stdout)
handler_file = RotatingFileHandler(
    'twitter_bot_monitor.log',
    mode='a',
    maxBytes=1048576,
    backupCount=9,
    encoding='UTF-8',
    delay=True
)
handler_file.setLevel(logging.DEBUG)
handler_file.setFormatter(formatter)
log.addHandler(handler_file)

path = os.path.dirname(os.path.abspath(inspect.getfile(inspect.currentframe())))

try:
    # read config
    config = configparser.SafeConfigParser()
    config.read(os.path.join(path, "config"))
except IOError:
    log.critical('configuration file is missing')
    sys.exit(0)

try:
    # your hashtag or search query and tweet language (empty = all languages)
    hashtag = config.get("settings", "search_query").split(' ')
    tweetLanguage = config.get("settings", "tweet_language")
    reply_text = config.get("settings", "reply_text").split(";")
    search_delay = int(config.get("settings", "search_delay"))
    search_depth = int(config.get("settings", "search_depth"))
    userBlacklist = [user.strip() for user in config.get("settings", "userBlacklist").split(",")]
    #TODO fix wordBlacklist
    #wordBlacklist = config.get("settings", "wordBlacklist").split(",")
    wordBlacklist = ["RT", u"♺"]
except Exception:
    log.critical('Please, fill the config file!')
    sys.exit(0)

# build savepoint path + file
hashedHashtag = hashlib.md5(hashtag[0]).hexdigest()
last_id_filename = "last_id_hashtag_%s" % hashedHashtag
rt_bot_path = os.path.dirname(os.path.abspath(__file__))
last_id_file = os.path.join(rt_bot_path, last_id_filename)

# create bot
try:
    auth = tweepy.OAuthHandler(config.get("twitter", "consumer_key"), config.get("twitter", "consumer_secret"))
    auth.set_access_token(config.get("twitter", "access_token"), config.get("twitter", "access_token_secret"))
    api = tweepy.API(auth)
except configparser.NoOptionError:
    log.critical('Problems with your login credentials!')
    sys.exit(0)

def wait_on_action():
    min_wait_time = 0
    max_wait_time = 0
    if config.get("settings", "min_wait_time"):
        min_wait_time = int(config.get("settings", "min_wait_time"))

    if config.get("settings", "max_wait_time"):
        max_wait_time = int(config.get("settings", "max_wait_time"))

    if min_wait_time > max_wait_time:
        temp = min_wait_time
        min_wait_time = max_wait_time
        max_wait_time = temp

    wait_time = random.randint(min_wait_time, max_wait_time)
    if wait_time > 0:
        log.info("Choosing time between %d and %d - waiting %d seconds before action\n"
                 % (min_wait_time, max_wait_time, wait_time))
        time.sleep(wait_time)
    return wait_time

def main():
    while True:
        log.info("Script is running! Please, wait.")
        # retrieve last savepoint if available
        try:
            with open(last_id_file, "r") as file:
                savepoint = file.read()
                log.info("Loaded savepoint from reply history. Starting search from tweet id %s\n" % savepoint)
        except IOError:
            savepoint = ""
            log.warning("No previously replied tweets found. Making a search request go get %d tweets!\n" % int(search_depth)*2)

        try:
            # search query
            timelineIterator1 = tweepy.Cursor(api.search, q=hashtag[0], since_id=savepoint, lang=tweetLanguage).items(search_depth)
            timelineIterator2 = tweepy.Cursor(api.search, q=hashtag[1], since_id=savepoint, lang=tweetLanguage).items(search_depth)
        except Exception:
            log.critical('Problem! Can t get search request!')
            sys.exit(0)

        # put everything into a list to be able to sort/filter
        timeline1 = []
        timeline2 = []

        for status in timelineIterator1:
            timeline1.append(status)

        for status in timelineIterator2:
            timeline2.append(status)

        results = set(timeline1 + timeline2)
        results = sorted(results, key=lambda x: x.id, reverse=True)

        try:
            last_tweet_id = results[0].id
        except IndexError:
            last_tweet_id = savepoint

        # filter @replies/blacklisted words & users out and reverse timeline
        results = filter(lambda status: status.text[0] != "@", results)
        results = filter(lambda status: not any(word in status.text.split() for word in wordBlacklist), results)
        results = filter(lambda status: status.author.screen_name not in userBlacklist, results)
        results.reverse()

        tw_counter = 0
        err_counter = 0
        dublicates = 0

        # iterate the timeline and retweet
        for status in results:
            try:
                log.info("LOADING. Proccesed %d of the %d tweets" % (tw_counter+dublicates+1, len(results)))
                if not os.path.exists("already_replied.txt"):
                    file = open('already_replied.txt', 'w')
                    file.close()
                with open("already_replied.txt", 'r+') as file:
                    replied_before = file.read()
                    if replied_before.find(str(status.id)) >= 0:
                        log.info("Skiped entry, because it s already replied!")
                        dublicates += 1
                        continue

                log.info("REPLIED TO THE TWEET")
                log.info("(%(date)s) %(name)s: %(message)s %(id)s\n" % \
                      {"date": status.created_at,
                       "name": status.author.screen_name.encode('utf-8'),
                       "message": status.text,
                       "id": status.id})

                try:
                    api.update_status(status="@" + status.author.screen_name.encode('utf-8') + " " + random.choice(reply_text),
                                  in_reply_to_status_id=status.id)
                except Exception:
                    log.critical("Can t update status!")
                    sys.exit(0)
                with open("already_replied.txt",'a') as file:
                    file.write(str(status.id) + '\n')
                wait_on_action()
                tw_counter += 1
            except tweepy.error.TweepError as e:
                # just in case tweet got deleted in the meantime or already retweeted
                err_counter += 1
                log.exception(e)
                continue
        log.info("Finished. %d Tweets retweeted, %d errors occured." % (tw_counter, err_counter))

        # write last retweeted tweet id to file
        with open(last_id_file, "w") as file:
            file.write(str(last_tweet_id))

        log.info("Script stops for %d seconds\n" % search_delay)
        time.sleep(search_delay)

if __name__ == '__main__':
    main()


---
layout: post
title:  "Release the bots"
tags: python note-to-self programming
thumb_nail: "/assets/photos/twitter.png"
text_excerpt: "Building a Twitter bot for Slack"
---

![](/assets/photos/twitter.png)
_"4: Bring me tweets."_

_― Matt's law of robotics_
<br><br>

As part of my plan to spend less time on-line in 2021, I decided to automate my [Twitter](https://twitter.com) scrolling. Being productive on Twitter means catching announcements, new papers, etc... from a small group of physicist and academics talking about esoteric concepts in science and technology. To help with this, I created a bot that streams tweets to a [Slack](https://slack.com) channel where I can discuss them with 'the bros'. I'm far from the first person to do this but hopefully there's some useful information here. Seems like this was _the_ thing to do in 2012, but no one has bothered to updated the syntax or authentication since. The rest of this post is a _highlights and how-to_ in the process of building the bot. If you'd like to skip to the punch line, the code is on github at [matthewware/slack-twitter-bot](https://github.com/matthewware/slack-twitter-bot). Things I will try to cover:

  * Authentication with Slack and Twitter
  * Streaming and filtering Twitter streams -- and catching errors
  * Slack message `Block` formatting and sending HTTP `POST` requests
  * Controlling the bot from Slack
  * Notes on deploying the bot in the cloud

> UPDATE 03/12/21: There were several issues in practice with this bot intermittently 
> crashing. After making several changes, I believe the major cause was the handling 
> of empty byte strings in python urllib3's `response.py`. See more 
> [here](https://github.com/psf/requests/issues/4248). After making the 
> `line = (len(line)>0 and line or "0")` change I haven't had any issues.

## Authentication

In total, we need six tokens for this bot to work. Two are for Slack and four are for Twitter.

### Twitter

Twitter handles its authentication on a developer basis. This allows them to attach API calls to the tokens of individual developers. On top of these, each bot/app you build will also have its own set of access tokens: 

  * `TWITTER_CONSUMER_KEY` - your individual developer key
  * `TWITTER_CONSUMER_SECRET` - your individual developer secret
  * `TWITTER_ACCESS_TOKEN` - an access token for your app itself
  * `TWITTER_ACCESS_TOKEN_SECRET` - an secret token for your app
 
To get started, go to the Twitter [developer portal](https://developer.twitter.com/en) and request a developer account. This isn't an automatic process. You'll need to _ask_ for an account and tell Twitter roughly what you plan to do with it. In my case, I only wanted to get data off Twitter and have no plans to post any of it publicly. That meant the process was pretty quick for me (\~5 minutes).

### Slack

For Slack all authentication is done on the applications/bot level. This means each bot will have a unique set of keys. For our purposes we only need a `SLACK_TOKEN` to talk to our app and a `SLACK_SIGNING_SECRET` so we can verify request from Slack to our control sever . The process for generating these is very simple and the first thing you do when you create your app/bot. The [Slack API documentation](https://api.slack.com/authentication/basics) is a great place to start learning. 

Every request to the Slack API will use the `SLACK_TOKEN`. A very basic usage of the `slack_sdk` would look something like this:
```python
new_token = os.environ['SLACK_TOKEN']
client = WebClient(token=new_token)

def write_text(message, channel='bot-dev'):
    try:
        response = client.chat_postMessage(channel=channel, text=message)
```

In the other direction, we want to verify request to the bot control server are legitimate. This [Flask](https://flask.palletsprojects.com/en/1.1.x/) server lets users update the _users_ and _keywords_ to follow on Twitter. The [documentation](https://api.slack.com/authentication/verifying-requests-from-slack) for checking these `HTTP` requests is pretty thorough. A `slack_sdk.verify.verify_request` function in the SDK will do the authentication for you. But I ended up implementing it myself because wanted to add a very simple timeout for request authentication:
```python
if abs(time.time() - int(timestamp)) > 60 * 5:
    # The request timestamp is more than five minutes from local time.
    # It could be a replay attack, so let's ignore it.
    return False
```

## Twitter streaming

The heart of the app is the Tweepy `Stream` object. It streams new tweets to your bot as they appear on Twitter. This is exactly what we want. The only issue is the `Stream` only lets you filter by user _or_ by keyword and not by both[^1]. It was far easier for me to filter first by users given I'm only planning to follow < 100 people. Given this and the volume these people tweet, the rest of the filtering is done by brute force. Your mileage may vary depending on who you follow and what keywords you want to track! Lastly, I filter out reply tweets just so I can focus on what people tweet to their accounts and not the arguments that follow.

The real interesting part of Tweepy streaming is setting up the stream in a way that makes it easy stop and restart when users and keywords are changed by the control server. To do that, I have the `launch_bot` and `restart_bot` functions: 

```python
def launch_bot(channel=POST_CHANNEL):
    """
    Start the stream and filter for users in the db list.
    All other filtering is done by the Listener.
    """
    logging.info("Creating listener...")
    myStreamListener = MyStreamListener(channel=channel)
    myStream = CustTweepyStream(auth = api.auth, 
                                listener=myStreamListener, 
                                include_entities=True, 
                                tweet_mode = 'extended')

    # start filtering
    logging.info("Starting bot...")
    # async needs to be true so we don't block the file watcher
    myStream.filter(follow=get_ids(), is_async=True, stall_warnings=False)

    return myStream

def restart_bot(stream):
  # try to kill previous stream
  stream.disconnect()
  # return a new stream but wait some time to avoid rate limiting
  time.sleep(60)
  return launch_bot()
```

This may look standard but you need to extend the `Stream` class by adding methods for `__enter__` and `__exit__` so the class can be used in *contexts*. This lets python know what to do when disconnect is called:
```python
class CustTweepyStream(Stream):
    """Custom Stream usable in contexts"""
    def finalize(self):
        pass

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.finalize()

    def __enter__(self):
        return self
```
Here we don't need these methods to do much, but we do need them to exist. For more on the reasons why, checkout this [blog post](https://www.pythonforthelab.com/blog/the-with-command-and-custom-classes/).

### Catching errors

Tweepy's connection to the Twitter API seems to suffer periodically from an IncompleteRead error.
This tends to happen roughly every \~day. There's some [speculation on StackOverflow](https://stackoverflow.com/questions/48034725/tweepy-connection-broken-incompleteread-best-way-to-handle-exception-or-can) this issue comes from getting too many tweets in a short amount of time. So that's pretty annoying. Current plans are to put the tweet data into a queue and process the queue, but that still needs more testing[^3]. For now I'm just catching the errors (and possibly missing a tweet here and there). Luckily, the `urllib3` and `http` libraries make it easy to catch this specific error. I found a discussion [here](https://github.com/tweepy/tweepy/issues/908) and added the following to the tweet processing code:

```python
from http.client import IncompleteRead as http_incompleteRead
from urllib3.exceptions import IncompleteRead as urllib3_incompleteRead
```
```python
# returning False closes the stream
except BaseException as e:
    logging.error("Error on_data: %s, Pausing..." % str(e))
    time.sleep(5)
    return True
except http_incompleteRead as e:
    logging.error("http.client Incomplete Read error: %s" % str(e))
    logging.error("~~~ Restarting stream search in 5 seconds... ~~~")
    time.sleep(5)
    #restart stream - simple as return true just like previous exception?
    return True
except urllib3_incompleteRead as e:
    logging.error("urllib3 Incomplete Read error: %s" % str(e))
    logging.error("~~~ Restarting stream search in 5 seconds... ~~~")
    time.sleep(5)
    return True
```

## Slack messages

The Slack documentation discourages the use of attachments given they are now deprecated. `Blocks` are now the blessed way to send rich message formats to a Slack channel. These are basically JSON objects that need certain fields. Getting the right message formatting was one of the more tedious parts of bot building. Luckily there's now a _Block Kit Builder_ [playground](https://api.slack.com/block-kit) for getting things just right. For the twitter bot, a `build_message` function parses the tweet object and fills in data for the block. Unfortunately, there didn't seem to be a way to avoid handling tweet, retweet, and quote tweet formatting all with separate block code.

## Control server

One feature I really wanted this bot to have is the ability to update Twitter users and keywords directly from the Slack channel. A simple control server allows the csv files to be updated from Slack via ['slash commands'](https://api.slack.com/interactivity/slash-commands). The server listens on a port and writes updated files to the local file system based on the verified commands from Slack[^2].

The tricky part of allowing slash commands is you'll need to stop and restart the Tweepy stream every time the users or keywords are updated. This disconnect / clean up / wait / reconnect process we covered a bit in the Streaming section. To trigger the process, there's a pretty nice file watching package call [`watchgod`](https://pypi.org/project/watchgod/) based on the [`watchdog`](https://pypi.org/project/watchdog/) package. It was pretty simple to extended the directory watcher to watch the keyword and user `.csv` files.

```python
class CSVWatcher(DefaultDirWatcher):
    def should_watch_file(self, entry):
        return entry.name.endswith(('.csv',))
```

When the file watcher sees a file change, it halts the current Tweepy stream and waits \~60 seconds before starting a new one to avoid the Twitter API rate limiting. In the main program, I have something like:

```python
bot_stream = launch_bot()

for changes in watch(os.path.abspath('.'), watcher_cls=CSVWatcher):
    bot_stream = restart_bot(bot_stream)
```

## Final thoughts

For large loads, take a look at the async Slack functions. These are nice if you expect a large volume of tweets and don't want your code to block. Then again... if all this is going to a Slack channel do you really want multiple messages per second coming to a channel? Here are some straight forward updates you might consider making:
   - maintain *keyword* and *user* files for multiple channels and have a context for each. This would allow the bot to join multiple channels and route certain tweets to certain channels.
   - write directly to multiple channels

### Deploying 

If you're planning to use this bot in 'production' I'd recommend the usual security procedures -- don't run as root etc... I have an instance running on DigitalOcean and setup should be relatively painless. You'll need to open the server control port in your firewall if you're using one (as you should be). If you expect lots of commands or run the Twitter stream in debug mode for a long time you might consider setting up a wsgi server like [waitress](https://docs.pylonsproject.org/projects/waitress/en/stable/) or [uwsgi](https://uwsgi-docs.readthedocs.io/en/latest/), using async functions, and doing [log rotation](https://www.tecmint.com/install-logrotate-to-manage-log-rotation-in-linux/).

---
<br>

[^1]: It's interesting to note the Twitter API allows this but the Tweepy implementation does not.
[^2]: I'd recommend testing and developing this server locally using [ngrok](https://dashboard.ngrok.com/get-started/setup).
[^3]: See the [feature/queue](https://github.com/matthewware/twitter-slack-bot/tree/feature/queue) branch for what I think is a working implementation. This is definitely the way to go if you have a large volume of tweets.
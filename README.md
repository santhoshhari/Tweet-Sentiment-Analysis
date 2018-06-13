# Twitter Sentiment Analysis

The goal of this project is to learn how to pull twitter data, using the [tweepy](http://www.tweepy.org/) wrapper around the twitter API, and how to perform simple sentiment analysis using the [vaderSentiment](https://github.com/cjhutto/vaderSentiment) library.  The tweepy library hides all of the complexity necessary to handshake with Twitter's server for a secure connection. Finally, produce a web server running on AWS to display the most recent 100 tweets from a given user and the list of users followed by a given user. For example, in response to URL `/the_antlr_guy` (`http://localhost/the_antlr_guy` when tested), the web server responds with a tweet list color-coded by sentiment, using a red to green gradient:

![](/figures/parrt-tweets.png)

As another example URL `/realdonaldtrump` yields:

![](/figures/trump-tweets.png)

I've also created a page responding to URLs, such as `/following/the_antlr_guy`, that displays the list of users followed by a given user:

![](/figures/parrt-follows.png)

Or:

![](/figures/trump-follows.png)

Note that the users are sorted in reverse order by their number of followers. Just to be clear, `/following/the_antlr_guy` shows the list of users that Terrance follows sorted by how many followers those users have. Clearly, Guido has the most followers and so he is shown first in my list of people I follow.

## Implementation Description

Using the twitter API, you can pull tweets and user information and then display using HTML.

### Authenticating with the twitter API server

Twitter requires that you register as a user and then also create an "app" for which Twitter will give you authentication credentials. These credentials are needed for making requests to the API server. Start by logging in to [twitter app management](https://apps.twitter.com/) then click on "create new app". It should show you a dialog box such as the following, but of course you would fill in your own details:

![](/figures/twitter-app-creation.png)

For the website, you can link to your LinkedIn account or something or even your github account. Leave the "callback URL" blank.

Once you have created that app, go to that app page. Click on the  "Keys and Access Tokens" tabs, which shows 4 key pieces that represent your authentication information:

* Consumer Key (API Key)
* Consumer Secret (API Secret)
* Access Token
* Access Token Secret

Under the Permissions tab, make sure that you have your access as "Read only" for this application. This prevents a bug in your software from doing something horrible to your twitter account!

**We never encode secrets in source code**, consequently, we need to pass that information into our web server every time we launch. To prevent having to type that every time, we will store those keys and secrets in a CSV file format:

*consumer_key*, *consumer_secret*, *access_token*, *access_token_secret*

The server then takes a commandline argument indicating the file name of this data. For example, I pass in my secrets via

```bash
$ sudo python server.py ~/Dropbox/licenses/twitter.csv
```

Please keep in mind the [limits imposed by the twitter API](https://dev.twitter.com/rest/public/rate-limits). For example, you can only do 15 follower list fetches per 15 minute window, but you can do 900 user timeline fetches.

### Mining for tweets

File `tweetie.py` (pronounced "tweety pie", get it?) has methods to fetch a list of tweets for a given user and a list of users followed by a given user.  Function `fetch_tweets()` returns a dictionary containing:

* `user`: user's screen name
* `count`: number of tweets
* `tweets`: list of tweets

where each tweet is a dictionary containing:

* `id`: tweet ID
* `created`: tweet creation date
* `retweeted`: number of retweets
* `text`: text of the tweet
* `hashtags`: list of hashtags mentioned in the tweet
* `urls`: list of URLs mentioned in the tweet
* `mentions`: list of screen names mentioned in the tweet
* `score`: the "compound" polarity score from vader's `polarity_scores()`

Function `fetch_following()` returns a dictionary containing:

* `name`: user's real name
* `screen_name`: Twitter screen name (e.g., `the_antlr_guy`)
* `followers`: number of followers
* `created`: created date (no time info)
* `image`: the URL of the profile's image

This information is needed to generate the HTML for the two different kinds of pages.

### Generating HTML pages

We can use the template engine [jinja2](http://jinja.pocoo.org/docs/2.9/) that is built-in with flask. When you call `render_template()` from within a flask route method, it looks in the `templates` subdirectory for the file indicated in that function call. You need to pass in appropriate arguments to the two different page templates so the pages fill with data.

Here is what a single tweet's HTML looks like:

```html
<li style="list-style:square; font-size:70%; font-family:Verdana, sans-serif; color:#ea4c00">
    -0.68: <a style="color:#ea4c00" href="https://twitter.com/the_antlr_guy/status/897491721944158208">RT @kotlin: Kotlin 1.1.4 is out! Auto-generating Parcelable impls, JS dead code elimination, package-default nullability &amp;amp; more: https://t.…</a>
</li>
```

## Launching your server at Amazon

Creating a server that has all the appropriate software can be tricky so I have recorded a sequence that works for me.

The first thing is to launch a server with different software than the simple  Amazon linux we have been using in class. We need one that has, for example, `numpy` and friends so let's use an *image* (snapshot of a disk with a bunch of stuff installed) that already has machine learning software installed: Use "*Deep Learning AMI Amazon Linux Version 3.1_Sep2017 - ami-bde90fc7*":

![](/figures/aws-ami.png)

Create a `t2.small` size computer (in Oregon; it's cheaper)!

When you try to connect, it will tell you to use user `root` but use `ec2-user` like we did for the other machines.  In other words, here's how I login:

```bash
$ ssh -i "parrt.pem" ec2-user@34.203.194.19
```

Then install software we need:

```bash
sudo pip install flask
sudo pip install tweepy
sudo pip install gunicorn
sudo pip install vaderSentiment
sudo pip install colour
```

Now, clone your repository into the home directory:

```bash
cd ~
git clone https://github.com/santhoshhari/Tweet-Sentiment-Analysis.git
cd sentiment-parrt
```

You should now be able to run your server:

```bash
$ gunicorn -D --threads 4 -b 0.0.0.0:5000 --access-logfile server.log server:app twitter.csv
```

(Test without `-D` during development so that you can see errors generated by the server; otherwise they appear to be hidden.)

`twitter.csv` is the file with your credentials.

All output goes into `server.log`, even after you log out. The `-D` means put the server in daemon mode, which runs the background.

Don't forget to open up port 5000 in the firewall for the server so that the outside world can access it. Make sure that you test from your laptop!

Make sure the `IP.txt` file as the **public** IP address of your server with `:5000` on the line by itself, such as `54.198.43.135:5000`!

**Note:** Project ideation and description credit goes to [Terrance](https://www.usfca.edu/faculty/terence-parr)

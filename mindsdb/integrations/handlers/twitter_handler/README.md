# Build your own twitter AI agent

Imagine that you want to engage with your followers on twitter and you want to answer all thier questions promptly.
Thanks to MindsDB. You can trian an AI system that help you manage automation on twitter responses.


For this particular example we would like to automatically respond to people that tweet about us as follows:
- If the message is positive, write a short thank you note. 
- If the message is is negative or the message has a question, invite them to our slack channel.


To do this we will build our own twitter AI tool in a few SQL commands in MindsDB.


Lets start by connecting our twitter account, to do that you can follow these [steps](https://developer.twitter.com/en/docs/authentication/oauth-2-0/bearer-tokens) to obtain a BEARER TOKEN from twitter.


```
# Should be able to create a twitter database
CREATE DATABASE my_twitter 
With 
    ENGINE = 'twitter',
    PARAMETERS = {
      "bearer_token": "twitter bearer TOKEN"
    };
```

This creates a database called my_twitter. This database ships with a tabled called tweets that we can use to search for tweets as well as to write tweets.


## Searching for tweets in SQL

Let's get a list tweets that contain or hashtag the word mindsdb

```
SELECT 
   id, created_at, author_username, text 
FROM my_twitter.tweets 
WHERE 
   query = '(mindsdb OR #mindsdb) -is:retweet -is:reply' 
   AND created_at > '2023-02-16' 
LIMIT 20;
```

MindsDB twitter integration also supports native queries, which in this case will be calling any funciton in tweepy
(https://docs.tweepy.org/en/stable/client.html)
```
# this should handle authentication and pagiantion for us
SELECT * FROM my_twitter (
  search_recent_tweets(
    query = '(mindsdb OR #mindsdb) -is:retweet -is:reply'',
    start_time = '2023-02-16T00:00:00.000Z',
    max_results = 2
  )
);
```

## Writing tweets using SQL

Let's test by tweeeting a few things

```
INSERT INTO my_twitter.tweets (reply_to_tweet_id, text)
VALUES 
    (1626198053446369280, 'MindsDB is great! now its super simple to build ML powered apps'),
    (1626198053446369280, 'Holy!! MindsDB is the best thing they have invented for developers doing ML'),
```

Those tweets should be live now on twitter, like magic right?

## Lets use AI to write responses for us

In order to do the task we have at hand we would like to create a machine learning model that can write responses. 
We will be using OpenAI GPT-3 for this, the way it works is that we create a model that can take a prompt and give as message based on that prompt, 
It looks like this:

```
CREATE MODEL mindsdb.twitter_response_model                           
PREDICT response
USING
  engine = 'openai', 
  max_tokens = 200,             
  prompt_template = 'from tweet "{{text}}" by "{{author_username}}", if their comment is a question invite them to join the MindsDB slack using this link http://bitly.com/abc. Otherwise, simply write a thank you message';
```

This created a virtual AI table called twitter_response_model, we can query this model as if it was a table, but it will generate respones for us, as follows:

```
SELECT response FROM mindsdb.twitter_response_model 
WHERE author_username = '@pedro' and text = 'I love this, can I learn more?';
```

Now, lets test it with some of the tweets in twitter, so we are going to join that model with the query that we had worked before that get the positive and neutral comments

```
SELECT t.id, t.author_username, t.text, r.response 
FROM my_twitter.tweets t
JOIN mindsdb.twitter_response_model r 
WHERE 
   t.query = '(mindsdb OR #mindsdb) -is:retweet -is:reply'' 
LIMIT 2
```
# At last: lets create the job

Finally, we can now automate the responses, by writing a job that:
- Checks for new tweets
- Generates a response using the OpenAI model
- Tweets the responses back 

All this in one  SQL command


```
CREATE JOB auto_respond AS (

 INSERT INTO my_twitter.tweets (in_reply_to_tweet_id, text)
 SELECT 
   t.id AS in_reply_to_tweet_id, 
   r.response AS text
 FROM my_twitter.tweets t
 JOIN mindsdb.twitter_response_model r 
      WHERE 
      t.query = '(mindsdb OR #mindsdb) -is:retweet -is:reply'' 
      AND t.created_at > "{{PREVIOUS_START_DATETIME}}"
  limit 2
)
EVERY HOUR
```

And there it is every hour, we will be checking for the new tweets created_at = {{PREVIOUS_START_DATETIME}}, and insert into tweets, the responses generated by OpenAI gpt 3


---
layout: post
title: Automated Tweets from Atom Feeds with Lambda
image: http://drive.google.com/uc?export=view&id=14a46XxEQtMl1t3Mh9-xDm4nsJLwz4nY3
---

As mentioned before, Troll Purse loves automation. One process at Troll Purse that was not automated was the Tweeting of Blog updates to the [Troll Purse Twitter account](https://www.twitter.com/trollpruse/). Orignally, Troll Purse used [Hoot Suite](https://www.hootsuite.com) to send out updates to multiple social channels at once. This was still a manual process that took time for something so repetitive. 

# Solving the Problem

While Hoot Suite is still useful, Troll Purse set up an environment using [AWS Lambda](https://aws.amazon.com/lambda/) and [AWS SNS](https://aws.amazon.com/sns/).

## The Design

![AWS Diagram for Auto Tweeting of Blog Updates](http://drive.google.com/uc?export=view&id=14a46XxEQtMl1t3Mh9-xDm4nsJLwz4nY3 "Auto Tweet Diagram in AWS")

The overall design is fairly simple. It is listed out in steps below:

1. An [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) Scheduled event is configured to execute on a [cron](https://en.wikipedia.org/wiki/Cron) schedule. 
2. The event is handled by a Lambda event that reads the data from the atom feed endpoint. 
3. The Lambda then converts the feed data from XML to JSON, gathering only the required information for the message.
4. The Lambda publishes this update to an [SNS](https://aws.amazon.com/sns/) topic.
5. A new Lambda is triggered by publishes to the specified SNS topic.
6. The new Lambda converts the `Body` payload of the SNS message into a Twitter tweet message.
7. The Lambda publishes the tweet using the [CoreTweet](https://github.com/CoreTweet/CoreTweet) Twitter library.

## The Lambda Codes

Writing code is always a joy. So, here is the code for the Atom Parsing to JSON Lambda. *Note:* This Lambda uses a Troll Purse internal library for the object, so if you are to copy this, you will need to define your own `BlogContentUpdated` and `ParsedAtomEntry` objects. For an example on how to parse RSS and Atom feeds, [this was a useful link to follow](http://www.anotherchris.net/csharp/simplified-csharp-atom-and-rss-feed-parser/).

```csharp
public async Task<string> FunctionHandler(dynamic input, ILambdaContext context)
{
    try
    {
        var request = await WebRequest.CreateHttp(FeedURL).GetResponseAsync();
        var latestAtom = XDocument.Load(request.GetResponseStream());
        var latestEntry = latestAtom.Root?
            .Elements()
            .FirstOrDefault(element => element.Name.LocalName.Equals("entry"));
        if (latestEntry == null)
        {
            return null;
        }
        var entry = new ParsedAtomEntry(latestEntry);
        var updateMessage = new BlogContentUpdated(entry.LinkHREF, entry.Title.Length > BlogContentUpdated.MaxTitleLength ? entry.Title.Substrin(0,BlogContentUpdated.MaxTitleLength) : entry.Title,
            Regex.Replace(entry.Content, "<.*?>", String.Empty).Substring(0, BlogContentUpdated.MaxSnippetLength));
        var publishRequest = new PublishRequest
        {
            TopicArn = Environment.GetEnvironmentVariable("topic_arn"),
            Message = JsonConvert.SerializeObject(updateMessage)
        };
        var response = await SNSClient.PublishAsync(publishRequest);
        return response.MessageId;
    }
    catch(Exception e)
    {
        context?.Logger.LogLine("Error with syndication.");
        context?.Logger.LogLine(e.Message);
        context?.Logger.LogLine(e.StackTrace);
        return null;
    }
```

With that. The Lambda triggered by the SNS notification uses a few short lines of code in the function definition.

```csharp
public async Task<string> FunctionHandler(SNSEvent input, ILambdaContext context)
{
    try
    {
        var messageJSONString = input.Records[0]?.Sns.Message;
        context?.Logger.LogLine($"Received({input.Records[0]?.Sns.MessageId}): {messageJSONString}");
        if (messageJSONString != null)
        {
            var magicCharCountForFormat = 7;
            var messageContent = JsonConvert.DeserializeObject<BlogContentUpdated>(messageJSONString);
            var snippetLength = MaxTweetLength - messageContent.PostTitle.Length - messageContent.PostLink.Length - magicCharCountForFormat - GameDevHashtags.Length;
            var tweetStatus = $"{messageContent.PostTitle}. {messageContent.ContentSnippet.Substring(0, snippetLength)}... {GameDevHashtags} {messageContent.PostLink}";

            var token = Tokens.Create(Environment.GetEnvironmentVariable("consumer_key"),
                Environment.GetEnvironmentVariable("consumer_secret"),
                Environment.GetEnvironmentVariable("auth_token"),
                Environment.GetEnvironmentVariable("auth_secret"));
            var response = await token.Statuses.UpdateAsync(tweetStatus);

            return response.CreatedAt.ToString();
        }
        else
        {
            return null;
        }
    }
    catch(Exception e)
    {
        context?.Logger.LogLine("Unable to Tweet SNS message");
        context?.Logger.LogLine(e.Message);
        context?.Logger.LogLine(e.StackTrace);
        return null;
    }
}
```

The majority of the work is done by the **CoreTweet** Library. Also notice that there are no hardcoded auth tokens - a good practice, especially since [environment variables for lambda can be stored as encrypted values](https://docs.aws.amazon.com/lambda/latest/dg/env_variables.html#env_encrypt).

# Core Tweet

Enough has been said about [Core Tweet](https://github.com/CoreTweet/CoreTweet) that it warrants it's own section. Setting up Core Tweet was extremely simple. It involved two multipart steps. The steps were to configure Twitter to enable app authorization for your account and install the [Core Tweet Nuget Package](https://www.nuget.org/packages/CoreTweet/) for [.Net Core](https://dotnet.github.io/).

Here are the steps simplified:

* Go to the [Twitter Apps website](https://apps.twitter.com/).
    * Log in to the Twitter account you want to have the Tweets pushed to.
    * Select *Create New App* Button.
    * Follow the Instructions
    * Under *Application Settings* Authorize the App the access the account you are logged into.
* [Setup your Lambda](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
    * Upload your function or create it inline
    * Save your keys and access tokens from Twitter as encrypted Lamnda Environment Variables

# Conclusion

Indeed, Troll Purse loves automation. Especially if it increases workflow and customer engagement. By automating the process of updating customers of new blog posts, Troll Purse is able to spend more time writing useful blog tutorials such as this and making games.

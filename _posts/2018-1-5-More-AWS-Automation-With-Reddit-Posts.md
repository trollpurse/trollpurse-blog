---
layout: post
title: More AWS Automation With Reddit Posts
image: https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/reddit-logo-banner.png
---

So many of these posts have been about process automation, so why stop! [Troll Purse](http://trollpurse.com) is at it again. This time, we are leveraging [AWS Lambdas](https://aws.amazon.com/lambdas/) to submit [dev blog](http://blog.trollpurse.com/) updates to the [Troll Purse Subreddit](https://www.reddit.com/r/trollpurse/). There is also an exciting twist: [.NET Core](https://dotnet.github.io/) was not the platform of choice, rather, [NodeJS](https://nodejs.org) was used.

![Reddit Image Logo](https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/reddit-logo-banner.png)

# About Languages in the Cloud

One of the most difficult concept to encourage when working in the cloud with teams is convincing developers that the language does not matter. Before the firestorm ensues, there are definitely benefits to using the same languages, but platforms aren't built on languages these days - rather frameworks and tools.

With this in mind, Troll Purse was able to avoid an issue of using raw HTTP to interact with the Reddit API for posting using OAUTH2. To Note, there is nothing wrong with this approach; however, Troll Purse developers love development speed and not wasting time reinventing the wheel.

# The Process and Soltuion

## First Crack with C#

There is a [Nuget Package](https://www.nuget.org/packages/redditsharp) to integrate .Net Core with a Reddit API; however, it only supports .NET Core 2.0 from nuget (or had no tags for .NET Core 1.0). This was a problem for integration with AWS Lamdas because the Lambdas only execute within a .NET Core 1.0 context. So, we had to move onwards.

## Second Try with Javascript

A nice fallback for writing AWS Lambdas and modules, due to the sheer amount of developers, is Javascript. Particulary, Javascript for [NodeJS](https://nodejs.org). Taking a scan of the [Reddit docs for useful APIs](https://github.com/reddit/reddit/wiki/API-Wrappers), Troll Purse came across [this one](https://www.npmjs.com/package/snoowrap). In two lines of code (in regards to the API calls) Troll Purse was able to automate [SNS](https://aws.amazon.com/sns/) receives and submit a link to the [Troll Purse Subreddit](https://www.reddit.com/r/trolpurse/).

### The Code ( YAY )

```javascript
var snoowrap = require('snoowrap');

exports.handler = function (event, context, callback) {

    if (event != null) {
        var record = event.Records[0];
        if (record != null) {

            console.log('Received(' + record.Sns.MessageId + '): ' + record.Sns.Message + ')');

            var body = JSON.parse(record.Sns.Message);

            const r = new snoowrap({
                accessToken: process.env.access_token,
                userAgent: process.env.user_agent,
                clientId: process.env.client_id,
                clientSecret: process.env.client_secret,
                refreshToken: process.env.refresh_token
            });

            var linkToSubmit = {
                title: body.PostTitle,
                url: body.PostLink
            };

            console.log("Sending: " + JSON.stringify(linkToSubmit));

            r.getSubreddit(process.env.subreddit)
                .submitLink(linkToSubmit)
                .then(result => callback(null, JSON.stringify(result || 'Success')));
        }
        else {
            console.log('null record');
            callback(null, null);
        }
    }
    else {
        console.log('No event object');
        callback(null, null);
    }
};
```

Again note how Troll Purse developers use secure environment variables to pass in sensitive or configuration data to Lambda functions.

# The End

All in all, when a team strips itself the constraint of being married to a language, so much more can get done at a better pace. Having to write an HTTP API Wrapper with OAUTH could have easily increased the development time by as much as four. By elimitating language prejudice, we were able to accomplish our goal in as little as 2 hours (including research). The age of the cloud eliminates the need to fight over picking a language.

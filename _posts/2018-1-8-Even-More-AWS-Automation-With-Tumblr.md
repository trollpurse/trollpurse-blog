---
layout: post
title: Even More AWS Automation With Tumblr
image: https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/tumblr-logo.png
---

Yes! Troll Purse has done it again! Looking at all the great social platforms out there, we were missing one that needed our posting! [Tumblr](https://www.tumblr.com) has now been integrated into our automation platform. Following our desire to automate a lot of our processes, our foray into blog sharing automation has brought us upon the beaches of Tumblr's API. Today, we again, share with you our implementation.

![Tumblr Logo](https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/tumblr-logo.png "Tumblr Logo")

# How We Did It

Just like our [Reddit](http://blog.trollpurse.com/More-AWS-Automation-With-Reddit-Posts/) link posting automation, Troll Purse decided to use NodeJS and a javascript API library for Tumblr, by Tumblr. The big reason behind this was because C-Sharp was not supported by the offical [Tumblr APIs](https://www.tumblr.com/docs/en/api/v2). Seeing that we already had 90% of the code to write another Javascript API, we went that route for the Tumblr API as well.

All the Troll Purse developers had to do was implement the same logic as we did for Reddit link posting. Then, we swapped out the client provider. This does say that some refactoring needs to be done as this was common logic copied for two different projects with the same parameters across. But, let us show you our first iteration that worked right off the bat! Again, note that we used *secured* environment variables for our configuration data!

## Ze Code
```javascript
var tumblr = require('tumblr.js');

var createTags = function (content) {
    var hashTagStr = 'indie,video games';
    var tagMap = {
        AWS: 'AWS,Amazon',
        UE4: 'UE4,Unreal Engine'
    }
    for (var tag in tagMap) {
        if (tagMap.hasOwnProperty(tag)) {
            var search = tagMap[tag].split(',');
            for (var i = 0; i < search.length; ++i) {
                var str = search[i];
                if (content.indexOf(str) >= 0) {
                    hashTagStr += ',' + tag;
                    break;
                }
            }
        }
    }
    return hashTagStr;
}

exports.handler = function (event, context, callback) {
    if (event != null) {
        var record = event.Records[0];
        if (record != null) {

            console.log('Received(' + record.Sns.MessageId + '): ' + record.Sns.Message + ')');

            var body = JSON.parse(record.Sns.Message);

            var client = tumblr.createClient({
                credentials: {
                    consumer_key: process.env.consumer_key,
                    consumer_secret: process.env.consumer_secret,
                    token: process.env.token,
                    token_secret: process.env.token_secret
                },
                returnPromises: true
            });

            var params = {
                title: body.PostTitle,
                url: body.PostLink,
                description: body.ContentSnippet + '...',
                tags: createTags(body.PostTitle + ' ' + body.ContentSnippet)
            };

            console.log("Sending: " + JSON.stringify(JSON.stringify(params)));

            client.createLinkPost(process.env.blog_name, params)
                .then(resp => callback(null, JSON.stringify(resp || 'Success')));
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

The one differece is that there is a `createTags` function. This stupid little function just scans our title and content snippet to apply predetermined hashtags that are relevant to the content. That way, Troll Purse is not just making another reposting spam bot with dozens of non-relevant hashtags.

# Farewell

Since this was pretty much a cut and dry repeat of our Reddit link posting, this was a short and sweet post. Here are all the resources we used to write this [Lambda](https://aws.amazon.com/lambda/) function for Tumblr posting:

* [Tumblr API Get Started](https://www.tumblr.com/docs/en/api/v2)
* [Creating a Tumblr App](https://www.tumblr.com/oauth/apps)
* [The Tumblr npm package](https://www.npmjs.com/package/tumblr.js)
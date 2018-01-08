---
layout: post
title: That Is Right, Discord Update Automation in AWS
image: https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/discord-logo.png
---

Recently, [Troll Purse](http://trollpurse.com) setup a publicly invite for the [Troll Purse Discord Server](https://discord.gg/bQ47YbF). And, as with all things, we decided to test out using [Discord Webhooks](https://support.discordapp.com/hc/en-us/articles/228383668-Intro-to-Webhooks) to push updates to our members in realtime. This is by far the most effective realtime pushing we have conceived yet. It was so easy, sharing it will be just as easy.

![Discord Logo](https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/discord-logo.png "Discord Logo")

# Using A Simple Webhook

Usually, the pattern at Troll Purse to push to third party accounts follows this pattern:

1. Sign up for the third party account
2. Register an application
3. Find an API wrapper library for said third party account
4. Publish an [AWS Lambda](https://aws.amazon.com/lambda/)
5. Post about it!

This time, we decided to skip step 3. For the most part, the developers at Troll Purse recognized that this push would require very little data transformation and authentication routines. In fact, all of the work was done in one `POST` request to the [Troll Purse Discord Server](https://discord.gg/bQ47YbF).

## The Code, Kind Human
```csharp
public async Task<string> FunctionHandler(SNSEvent input, ILambdaContext context)
{
    try
    {
        var messageJSONString = input.Records[0]?.Sns.Message;
        context?.Logger.LogLine($"Received({input.Records[0]?.Sns.MessageId}): {messageJSONString}");
        if (messageJSONString != null)
        {
            var messageContent = JsonConvert.DeserializeObject<BlogContentUpdated>(messageJSONString);
            using (var httpClient = new HttpClient())
            {
                string payload = $"{{\"content\":\"{messageContent.PostTitle}. {messageContent.ContentSnippet}... {messageContent.PostLink}\"}}";
                var response = await httpClient.PostAsync(Environment.GetEnvironmentVariable("discord_webhook"), new StringContent(payloadEncoding.UTF8, "application/json"));
                return response.StatusCode.ToString();
            }
        }
        else
        {
            return null;
        }
    }
    catch (Exception e)
    {
        context?.Logger.LogLine("Unable to Discord the SNS message");
        context?.Logger.LogLine(e.Message);
        context?.Logger.LogLine(e.StackTrace);
        return null;
    }
}
```
*Notes:*
* `BlogContentUpdated` is code defined in an external Troll Purse binary.
* WE USE SECURE ENVIRONMENT VARIABLES!!! THIS IS IMPORTANT!!!! (As opposed to plaintext credentials in our source code.)

## The Joy of Lambda

All of these features that Troll Purse has blogged about are done within a few hours. This is easily aided by the idea of serverless programming. There is no overhead of provisioning servers, testing different server environments, and configuring a network for these functions. It removes a lot of network infrastructure and enables Troll Purse developers to create fast, reactive, internal services.

Please, if you spend too much time configuring and setting up, try using [AWS Lambda](https://aws.amazon.com/lambda/) to speed up development time.

# Would You Look At That

In two lines, without a library or API wrapper, our developers can now push blog updates to our Discord server. This is a nice quick feature that we plan on integrating in our automated build environment to push updates about new versions released to the public. Enjoy!
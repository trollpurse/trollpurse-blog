---
layout: post
title: Testing AWS Lambdas
image: https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/lambda-icon.png
---

Recently, Troll Purse had issues with the [Auto Tweet Lambda](http://blog.trollpurse.com/Automated-Tweets-from-Atom-Feeds-with-Lambda/). While not our fault in the end, it took local debugging and tests to verify. Sharing is caring, here is one way to locally test and debug [.Net Core](https://dotnet.github.io/) [AWS Lambda](https://aws.amazon.com/lambda/).

![AWS Lambda Icon](https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/lambda-icon.png "AWS Lambda Icon")

# Unit Tests

Unit tests are an excellent way to verify class and service code - no matter the language you are writing in. So, naturally, Troll Purse added a suite of automated Unit Tests for AWS Lambdas. This is no different than any other [C# Unit Test](https://docs.microsoft.com/en-us/visualstudio/test/create-unit-tests-menu).

In fact, the [AWS Visual Studio Extension](https://aws.amazon.com/visualstudio/) allows you to create a [Lambda with Unit Tests](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/lambda-index.html) in place!

# Debugging

While unit tests are nice, issues happen - as Troll Purse just discovered today! While the issue was malformed data from the Atom feed, it still required some debugging. Naturally, Lambda functions are compiled as [DLLs](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682589(v=vs.85).aspx), which makes it hard to execute standalone to step through the code during runtime.

Thankfully, the unit tests can be executed in debug mode from Visual Studio which runs the Lambda. Using this knowledge, Troll Purse developers found a relevant test that was failing under the same conditions (invalid feed format) and debugged the production Atom feed from the Test Suite. This gave the developers the ability to verify where the [Exception](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/exceptions/) was occuring.

# Closing Thoughts

Test and Automation are a truly wonderous marvel in this day and age of software developement. Thankfully, the developers at AWS agree and created a neat tool and framework to aid in the speedy development of tests for Lambdas.
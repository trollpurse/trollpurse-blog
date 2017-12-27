Recently, Troll Purse made the decision to migrate from the cloud in Digital Ocean to Amazon Web Services for cloud and website services. There were several reasons behind this critical decision. These reasons are infrastructure, flexibility, and future plans.

![Image of AWS logo and Digital Ocean logo, side by side.](https://raw.githubusercontent.com/trollpurse/trollpurse-blog/master/images/aws-v-do.png "AWS over Digital Ocean")

## Infrastructure

Digital Ocean has a nice setup. They have a slick looking User Interface, easy to find services, and awesome community driven documentation. Troll Purse quite liked Digital Ocean if not for a few issues regarding infrastructure in comparison to Amazon Web Services (AWS).

### Cost

Digital Ocean was not overly expensive. However, their base monthly rate ($5 / month + $1 / month for backups), was still higher than hosting a static website on AWS. AWS charges $0.50 / month for Route 53 (DNS Name Servers) and a variable cost for S3 storage of our static website content. Based on traffic trends, the variable costs of AWS were a huge benefit. Finally, it helps that AWS offers a full year of various resources for free.

### Continuous Delivery

Digital Ocean had a lot of APIs for Continous Delivery of our website, but it didn't offer a full suite of solutions for proper Developer Operations. Digital Ocean would require a lot of extra Developer Operations overhead writing build and deployment scripts using their APIs. AWS integrates with Bitbucket and Github - two services we use for source control. AWS also offers managed build and deployment services that Troll Purse will be leveraging.

### Architecture

Digital Ocean is limited to [Droplets](https://www.digitalocean.com/products/compute/). This pales in comparison to AWS's robust [EC2](https://aws.amazon.com/ec2/), [ECS](https://aws.amazon.com/ecs/), or [serverless](https://aws.amazon.com/serverless/) services. In AWS, Troll Purse can decouple services and code for various solutions. To do the same distributed computing in Digital Ocean as Troll Purse is enabled to do in AWS would require a significant investment in architecting infrastructure. In AWS, this is vastly done for Troll Purse - once an architecture is designed, only configuration of the services need to be done.

With Digital Ocean, Troll Purse had to setup an Nginx server for serving static website content. It did not scale well as all content was stored on that server. In AWS, Troll Purse can distribute web content using Cloud Front for CDN and server static content from S3. Totally "serverless" and decoupled from the blogging platform and the forum servers (the latter is yet to be deployed).

## Flexibility

In Digital Ocean it was difficult to manage servers and logical groupings of services. AWS offers tagging of resources and services to help better monitor health, costs, and grouping of application services built and provided by Troll Purse.

Unfortunately, Digital Ocean does not offer the robust services and infrastructure needed for a company built with development speed in mind. A lot of services need to be hand crafted, as well as servers. This slows development efforts and makes iterative development a nightmare. Using small, simple services each with a specific purpose in mind, Troll Purse is better able to develop an experiment based on the offerings provided by AWS.

Digital Ocean only provides five real services, Compute, Object Storage, Block Storage, Networking, and Monitoring. Each of these are small in comparison to comparable AWS services such as [Compute Services (ECS or EC2)](https://aws.amazon.com/products/compute/), [Storage](https://aws.amazon.com/products/storage/), [Virtual Private Cloud](https://aws.amazon.com/products/networking/) (Networking), and [Cloud Watch](https://aws.amazon.com/cloudwatch/) (Monitoring). In AWS each of those services / categories work with each other or are umbrellas to a myriad of other flexible offerings.

Finally AWS has way more to offer - [just take a look](https://aws.amazon.com/)!

## Future Plans

After much discussion, Troll Purse concluded our future development needs will be implemented faster and scale naturally in AWS over Digital Ocean. While we enjoyed our brief experience with Digital Ocean, we are excited to build using AWS as our cloud and hosting provider.

While we do not plan to build anything with Lumberyard now, we have some exciting projects that will easily leverage the power of AWS. Such projects are backend analytics of our games using [Amazon API Gateway](https://aws.amazon.com/api-gateway/), [Lambda](https://aws.amazon.com/api-gateway/), and [data services](https://aws.amazon.com/products/databases/). We also hope to build an integrated environment so that each game shares common interfaces into our development and publishing environment. 

## Conclusion

While this blog post is mostly given as an explanation of our tactical decision to migrate to AWS, Troll Purse hopes it will serve as guidance to future developers facing the same decisions. We found that AWS provides a lot for small and large companies when it comes to infrastructure, flexibility, and accommodating future plans.
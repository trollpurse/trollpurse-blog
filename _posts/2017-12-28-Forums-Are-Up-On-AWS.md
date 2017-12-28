---
layout: post
title: Forums Are Up On AWS
---

After our migration to [AWS](https://aws.amazon.com), [Troll Purse](http://trollpurse.com) removed the old forums running in [Digital Ocean](https://www.digitalocean.com). Troll Purse decided to start with a clean slate. Which was easy - as nobody registered (no migrations needed, just nuclear destruction of the service)! A curse that turned into a blessing. Troll Purse can now scale the [forums](http://forums.trollpurse.com) based on usage and save some money on infrastructure. This will allow us to put more effort into our games!

![NodeBB Logo](https://community.nodebb.org/assets/uploads/system/site-logo.png "NodeBB Logo")

## How To

Troll Purse decided to share with you how to set up this type of environment.

### S3 Configuration

For hosting content uploaded by Troll Purse forum users, [S3](https://aws.amazon.com/s3/) was used to store images. Since [NodeBB](https://nodebb.org/) has a nice [S3 upload plugin](https://github.com/louiseMcMahon/nodebb-plugin-s3-uploads), there was little to no work other than configuration needed to enable the feature.

S3 on the otherhand, required configuration to allow access from http://forums.trollpurse.com. However, it also needed to allow access to the real DNS hostname (according to AWS) for the actual server to update data. This meant a custom [S3 CORS policy](http://docs.aws.amazon.com/AmazonS3/latest/dev/cors.html) and [S3 Bucket Policy](http://docs.aws.amazon.com/AmazonS3/latest/dev/example-bucket-policies.html). Finally, the role our server would assume needed to have full access to S3 buckets. Further, Troll Purse could restrict access by bucket name.

Below are examples Troll Purse Built up to help restrict access to an S3 bucket. Note, AWS will still mark it as public. However, there was a configuration that allowed public GET without S3 being marked public.

#### S3 Bucket Policy

~~~json
{
    "Version": "2012-10-17",
    "Id": "website access bucket Policy",
    "Statement": [
        {
            "Sid": "Allow get requests originating from your domain.",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-bucket-name/*",
            "Condition": {
                "StringLike": {
                    "aws:Referer": "http://your-domain-name/*"
                }
            }
        },
        {
            "Sid": "Deny get requests not originating from your domain.",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-bucket-name/*",
            "Condition": {
                "StringNotLike": {
                    "aws:Referer": "http://your-domain-name/*"
                }
            }
        },
        {
            "Sid": "Create, Update, Delete for ARN",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::xxxxxxxxx:role/your-role-used-for-s3-access-and-management"
            },
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::your-bucket-name/*"
        }
    ]
}
~~~

#### S3 CORS Policy

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
<CORSRule>
    <AllowedOrigin>your-domain-name</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>POST</AllowedMethod>
    <AllowedMethod>DELETE</AllowedMethod>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
    <AllowedHeader>Authorization</AllowedHeader>
</CORSRule>
</CORSConfiguration>
~~~

### Redis Configuration

For [Redis](https://redis.io/), Troll Purse used default configurations provided by [AWS ElastiCache](https://aws.amazon.com/elasticache/). This cache was put in a private subnet, accessible only to services in the Troll Purse VPC as configured. Currently, Troll Purse is using the free tier `cache.t2.micro` instance. Other than that, the Launch Configuration just needs a reference to the public DNS of the cache.

### VPC Configuration

[AWS VPC](https://aws.amazon.com/vpc/) is great for creating logically segregated services for an environment.

#### Subnets

Following normal AWS architecture diagrams (*shown below*), Troll Purse created two [subnets](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html#what-is-vpc-subnet). There is the public subnet which will host the forum instances and the load balancer. There is then the private subnet which has no internet access. The private subnet contains the forum's Redis service.

![AWS VPC Architecture Diagram](https://docs.aws.amazon.com/quickstart/latest/vpc/images/quickstart-vpc-design-fullscreen.png "AWS VPC Architecture Diagram")

#### Security Groups

Troll Purse setup two different [Security Groups](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html). One for services bound to the public subnet and another for services bound to the private subnet. The only real different is how the inboud internet traffic is configured. The public security group allows inbound internet traffic. The private security group does not allow inbound internet traffic. This is further strengthened by Route Tables

#### Route Tables

The Route Tables used were configured according to the afore mentioned diagram. There were two [Route Tables](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Route_Tables.html). The first route table was created for the public subnet. This allows internet traffic in via the [Internet Gateway](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html) bound to the public subnet. The second route table created was the private subnet. This Route Table did not receive configuration for public internet access.

### IAM Role Configuration

To get our environment up using NodeBB with Redis, Troll Purse created a new [IAM Role](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) for EC2 instances meant to host NodeBB. This role did not need a lot of thought put into it. All it needed was full S3 Access, and full Redis access. From here Troll Purse uses two more AWS services to provide data storage for the forums.

### Auto-Scaling Configuration

Using our existing configuration, Troll Purse created an [Auto Scaling](https://aws.amazon.com/autoscaling/) Configuration using the base Amazon Linux AMI on a `t2.micro` instance. We don't do anything else special. Troll Purse set the default configurations of Min instances to 1 and Max instances to 1. This ensures the service will always be running one instance, whether it fails or not.

*Note: Make sure to use ELB Health Checks - this will verify the web service is actually running on the instance*

#### Launch Configuration User Data

Here is a wonderful gist provided by one of our AWESOME developers (*Disclaimer:* I authored this post - totally biased opinion) used as a Launch Configuration.

<script src="https://gist.github.com/hollsteinm/bb6b0dc5632d160b4198b947cdb493aa.js"></script>

Soon Troll Purse will take away half of that setup and make an image for EC2 to use. Then only NodeBB configuration and launch information is required for the Launch Configuration.

### EC2 Configuration

There wasn't anything to do for [EC2](https://aws.amazon.com/ec2/) since all of our instance information was setup using Auto Scaling.

## Conclusion

Setting up an environment in AWS for our forums took about two days of building and verifying. These changes required no code whatsoever. All Troll Purse had to do was select from a large suite of services to support desired results. So, now that they exsist, [join up on the forums!](http://forums.trollpurse.com/register)
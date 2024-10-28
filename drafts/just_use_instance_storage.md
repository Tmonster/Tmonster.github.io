---
layout: post
title: Use Instance storage whenever you can
gh-repo: Tmonster/tmonster.github.io
gh-badge: [star, fork, follow]
tags: [Cloud Systems]
comments: true
---

Early in my computing days, when I was still a newbie when it came to AWS, I thought AWS was just a way to rent computers in the cloud. You could choose a specific configuration with a storage size and get to work. I didn't do any more research into what you could do with EC2. Most of the time I would choose a cheap instance with 2 CPU cores and 8GB of memory and be proud of myself that I could access a computer in "the cloud". Since this micro instance was on the free tier, I didn't think I would have to pay any money. This was very naive, however, as EBS storage **does** cost money. Still being a newbie to computer science, I was also very hesitant to throw away data, thinking maybe I left some important document there that I didn't back up anywhere else. After paying a high AWS bill as a broke college student, I doubled checked to make sure the EBS instance didn't have any data that wasn't backed up anywhere else, and I deleted the instance. 

working on AWS EC2 instances, I just assumed every system needed to be backed by EBS. The default size for every instance is 8GB, but of course I would need more, so I bumped it up to 50GB or 100GB. What took me a while to understand, unfortunately, is that EBS is **very expensive**, and instance storage is free when the machine is on. Instance storage gets wiped when the instance is shut down, but you can always upload whatever data you want persisted back to s3 if you need to.

Let's look at some scenarios and try to understand why a large EBS instance would be needed.

Basically, why are we always using EBS?


An EC2 instance with large EBS backed storage can be nice, but if are not careful, you could end up paying something like $XXX per month for your EBS backed instances.


An EC2 instance that has instance storage does not have this problem. If I rent a c6id.4xlarge, I get 800 GB of instance storage for free. This storage is also not network throttled. If I run a benchmark using an EBS backed data vs c6id.4xlarge vs a storage backed data, I can get the following results. 

But then where do I put my data? 
On s3, duh.

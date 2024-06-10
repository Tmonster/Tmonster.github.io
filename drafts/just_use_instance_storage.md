

Basically, why are we always using EBS?

An EC2 instance with large EBS backed storage can be nice, but if are not careful, you could end up paying something like $XXX per month for your EBS backed instances.


An EC2 instance that has instance storage does not have this problem. If I rent a c6id.4xlarge, I get 800 GB of instance storage for free. This storage is also not network throttled. If I run a benchmark using an EBS backed data vs c6id.4xlarge vs a storage backed data, I can get the following results. 

But then where do I put my data? 
On s3, duh.

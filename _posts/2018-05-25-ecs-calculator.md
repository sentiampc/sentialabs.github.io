---
layout: post
title: Cost efficiency while using the containers
banner: /assets/posts/2018-05-25-ecs-calculator/ecs-banner.png
author:
  - veranikaisakova
  - hadoan
---

In the cloud, containers provide a containerised environment enabling your code to be built, shipped and run anywhere. This can be simply done by just running your code without setting up your operating system.

Coming with such benefit, AWS Elastic Container Services (ECS) is a a highly scalable, fast, container management service that makes easy to run your containerised code and applications across a managed cluster of EC2 instances. You will need to optimally scale and fit ECS containers in an EC2 instance for the efficient usage of your resources in the cloud. This means you should ensure that resources such as CPU and Memory units are enough and efficiently reserved for your largest container to scale out.
The table below provides the values of the CPU and Memory reservation for each EC2 instance type as inputs to how to optimally fit ECS containers in an EC2 instance. Furthermore, how this is achieved will be explained in the latter examples.

  <div markdown="1" class="table-responsive ec2-table">

  |Instance<br/>Type| CPU| RAM|
  | :-------------: | :----: | :----: |
  |t2.micro| 1024| 993|
  |t2.small| 1024| 2001|
  |t2.medium| 2048| 3952|
  |t2.large| 2048| 7984|
  |t2.xlarge| 4096| 16048|
  |t2.2xlarge| 8192| 32176|
  |----------------------|
  |m4.large| 2048| 7984|
  |m4.xlarge| 4096| 16048|
  |m4.2xlarge| 8192| 32176|
  |m4.4xlarge| 16384| 64417|
  |m4.10xlarge| 40960| 161185|
  |m4.16xlarge| 65536| 257953|
  |----------------------|
  |m3.medium| 1024| 3765|
  |m3.large| 2048| 7480|
  |m3.xlarge| 4096| 15040|
  |m3.2xlarge| 8192| 30160|
  |----------------------|
  |c4.large| 2048| 3765|
  |c4.xlarge| 4096| 7480|
  |c4.2xlarge| 8192| 15040|
  |c4.4xlarge| 16384| 30145|
  |c4.8xlarge| 36864| 60385|
  |----------------------|
  |c3.large| 2048| 3765|
  |c3.xlarge| 4096| 7480|
  |c3.2xlarge| 8192| 15040|
  |c3.4xlarge| 16384| 30145|
  |c3.8xlarge| 32768| 60385|
  |----------------------|
  |r4.large| 2048| 15292|
  |r4.xlarge| 4096| 30664|
  |r4.2xlarge| 8192| 61408|
  |r4.4xlarge| 16384| 122881|
  |r4.8xlarge| 32768| 245857|
  |r4.16xlarge| 65536| 491809|
  |----------------------|
  |r3.large| 2048| 15299|
  |r3.xlarge| 4096| 30680|
  |r3.2xlarge| 8192| 61442|
  |r3.4xlarge| 16384| 122950|
  |r3.8xlarge| 32768| 245996|
  |----------------------|
  |i2.xlarge| 4096| 30680|
  |i2.2xlarge| 8192| 61442|
  |i2.4xlarge| 16384| 122950|
  |i2.8xlarge| 32768| 245996|
  |----------------------|
  |i3.large| 2048| 15292|
  |i3.xlarge| 4096| 30664|
  |i3.2xlarge| 8192| 61408|
  |i3.4xlarge| 16384| 122881|
  |i3.8xlarge| 32768| 245857|
  |i3.16xlarge| 65536| 491809|
  |----------------------|
  |m5.large| 2048| 7690|
  |m5.xlarge| 4096| 15586|
  |m5.2xlarge| 8192| 31377|
  |m5.4xlarge| 16384| 62960|
  |m5.12xlarge| 49152| 189294|
  |m5.24xlarge| 98304| 378665|
  |----------------------|
  |c5.large| 2048| 3714|
  |c5.xlarge| 4096| 7634|
  |c5.2xlarge| 8192| 15473|
  |c5.4xlarge| 16384| 31152|
  |c5.9xlarge| 36864| 70351|
  |c5.18xlarge| 73728| 140780|
  |----------------------|
  |d2.xlarge| 4096| 30664|
  |d2.2xlarge| 8192| 61408|
  |d2.4xlarge| 16384| 122881|
  |d2.8xlarge| 36864| 245857|

  </div>

❗️You can also find this information in yaml format [here](/assets/files/2018-05-25-ecs-calculator/ec2-instance-list.yml) 👈

The solution is to make the optimal choice of the EC2 instance type with its budgeted CPU and Memory units to efficiently fit ECS containers in the instance based on the data from the table. Therefore, it is important to first have some insight into the limits of CPU and Memory reservation.
The CPU units can be reserved with soft limit because containers can burst above their provision. One container can burst above its allocated units if no other containers are taking up resources. Besides, containers also share their unallocated CPU units with other containers on the instance with the same ratio as their allocated amount.
Memory part is here!!!!!!
=======
How the data can help you to optimally scale and fit ECS containers in an EC2 instance?
==========================================================================================

Based on the soft constraint/hard constraint of CPU/Memory reservation, the approach of solving our current issue is taken according to the following steps.
* Obtaining the number of the Memory units required for your largest container.
* Optimally choosing the EC2 instance type that can budget those Memory units.
* Calculating CPU units needed for your largest container.
The related example is given below.
Assume that your largest container requires 332 memory units. If you choose `t2.micro` as the EC2 instance type which offers 993 memory units, you can fit 2 containers in the instance (993/332 = 2.99 containers). However, if you choose `t2.small` which offers 2001 memory units, the memory resource would be more efficiently used because 2001/332 = 6.02 containers and not using the leftover 0.02 containers would be less wasteful than doing this with the leftover 0.99 containers. In this way, your containers fit more efficiently in the `t2.small` instance than in the `t2.micro` instance.
Calculate CPU units!!!
In another scenario, your largest container is provided with 1750 memory units. You can fit 1 container in the `t2.small` instance which reserves 2001 Memory units (2001/1750 = 1.14 containers). Else, the `t2.medium` instance which reserves 3952 Memory units makes it possible for 2 containers to fit in (3952/1750 = 2.25 containers). Then your containers will fit more efficiently in the `t2.small` instance than in the `t2.medium` instance with the same argument from the previous example applied.
Calculate CPU units!!!
The examples above are indeed based on the important idea: available resources should be used up for the sake of the efficiency. Therefore, in case you have troublesome after some time to find an EC2 instance type in the table that suits your needs, there are two options being advised:
* Providing more Memory units if available to your largest container so that the portion of your unused resources will be efficiently decreased.
* Using the `m4.large` instance for it overall reserves good amount of CPU/Memory units (2048/7984) for your containers and is used for general purposes.



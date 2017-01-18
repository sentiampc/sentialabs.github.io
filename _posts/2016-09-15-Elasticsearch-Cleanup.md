---
layout: post
title: Cleanup ElasticSearch Indexes
banner: https://static-www.elastic.co/assets/blt6050efb80ceabd47/elastic-logo%20(2).svg?q=200
author: svdgraaf
---

Sometimes, you just want to cleanup old ElasticSearch indexes, especially logstash indexes can become quite numerous.

Deleting everything from a certain month:

```bash
curl -XDELETE http://127.0.0.1:9200/logstash-2016.6.[1-30]
```

Or multiple months:

```bash
curl -XDELETE http://127.0.0.1:9200/logstash-2016.[1-6].[1-30]
```

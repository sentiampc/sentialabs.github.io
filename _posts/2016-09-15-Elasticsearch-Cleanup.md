---
layout: post
title: Cleanup ElasticSearch indexes
author: Sander van de Graaf
email: sander.van.de.graaf@sentia.com
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

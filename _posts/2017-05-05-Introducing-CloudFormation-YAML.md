---
layout: post
title: Introducing CloudFormation Resource YAML
banner: /assets/cloudformation/cloudformation_yml.png
author: thomas
belongs_to: cloudformation.md
---
At SENTIA we have developed our own object model for CloudFormation templates.
For the generation of the CloudFormation Resource object in this object model,
we parse the CloudFormation [Resource Types Reference].

Given that there are many inconsistencies in the HTML of the documentation,
parsing this documentation is non-trivial. Therefore, we decided to generate
a YAML file for public use, so that others can profit from the effort we put in
maintaining the parser.

Find it on GitHub:
[sentialabs/cloudformation](https://github.com/sentialabs/cloudformation)

[Resource Types Reference]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
---
title: 一个无聊的网站
date: 2023-09-14T16:47:12+08:00
tags: ["boring", "AWS", "website"]
series: ["Boring website"]
featured: true
---

本文是无聊网站的第一篇，主要介绍了网站起因和网站架构。

<!--more-->

### 建站起因

很久之前，在网上看到过一篇文章，讲述了一个无意义的网站。这个网站只有一个按钮，一个倒计时和一行文字。如果用户在一定的时间内（也没有具体说明是多久）不点击这个按钮，该网站将永远关闭，据说到发文为止，网站也不曾关闭。可惜，我从没有找到这个网站。但是某一天当我又想起这个事情的时候，我突然意识到，我自己可以去验证这个事情。所以就申请了一个域名[freeupdateme.com](https://freeupdateme.com)，准备搭建一个类似的网站。但是又不想仅仅只放一个按钮，所以添加了一个输入框和更新计时器，这样也可以拿到两次输入的间隔时间用来做验证。

### 网站功能

为了增加趣味性，我利用[ip2region](https://github.com/lionsoul2014/ip2region)解析IP地址获得地址信息，并且还添加了今日更新次数和总更新次数。为了防止输内次数过多，我对输入作了限制，当前每天仅限输入1000次。
在积累了大概八百条记录时候，添加了词云图，使用[结巴中文分词](https://github.com/fxsjy/jieba)，图片绘制使用[wordcloud2](https://wordcloud2-js.timdream.org/#love)在线生成。

{{< figure src="/images/blog/website-screenshot.png" height="500">}}

### 网站架构
网站部署在AWS云上，主要服务有API Gateway、Lambda、DynamoDB和S3，这些都是AWS上的基础服务，架构如下图。

{{< figure src="/images/blog/boring-website-architecture.png">}}

### 技术难点

很多使用AWS的朋友可能会被VPC等基础配置所困扰，但是这个网站可以不使用VPC和Subnet等基础配置。最开始我也走了弯路，将Lambda部署在default VPC中的public subnet中，又配置了一个dynamoDB的VPC endpoint才使得Lambda能够访问DynamoDB。但是因为Lambda不需要访问VPC内资源，所以可以不配置任何VPC。浏览器可以直接访问S3是因为配置了一个公开访问的Bucket资源，主要用来链接网站上的两张图片。

### 参考资料
1. [what-is-the-downside-of-not-running-aws-lambda-functions-in-a-vpc](https://stackoverflow.com/questions/45580610/what-is-the-downside-of-not-running-aws-lambda-functions-in-a-vpc)
2. [why-cant-an-aws-lambda-function-inside-a-public-subnet-in-a-vpc-connect-to-the- internet](https://stackoverflow.com/questions/52992085/why-cant-an-aws-lambda-function-inside-a-public-subnet-in-a-vpc-connect-to-the)


---
title: 使用JMeter对API进行压力测试
date: 2023-12-15T17:00:00+08:00
tags: ["JMeter", "测试", "API"]
series: ["测试"]
featured: true
---

本文介绍了JMeter，一款Java编写的API压力测试工具。以访问wikipedia主页为例，图文介绍了如何构建测试用例。

<!--more-->

## JMeter简介
Apache JMeter 是Apache组织的开放源代码项目，是一个纯Java桌面应用，用于压力测试和性能测试。

## 下载与启动
因为JMeter是Java桌面应用，所以需要提前安装并配置好Java环境。博主使用的是Mac，所以下载[apache-jmeter-5.6.2.tgz](https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.2.tgz).
下载完成后双击解压，进入`bin`目录，使用`./jmeter.sh`启动软件

## 创建测试
### Thread Group
在`Test Plan`右键 --> `Add` --> `Threads(Users)` --> `Thread Group`
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Create Thread Group.png">}}

Thread Group配置
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Thread Group Config.png">}}

### HTTP Request
在`Thread Group`右键 --> `Add` --> `Sampler` --> `HTTP Request`
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Create HTTP Request.png">}}

HTTP Request配置
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/HTTP Request Config.png">}}

### Response Assertion
在`Thread Group`右键 --> `Add` --> `Assertionis` --> `Response Assertion`
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Create Response Assertion.png">}}

Response Assertion Config配置
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Response Assertion Config.png">}}

### View Results Tree
在`Thread Group`右键 --> `Add` --> `Listener` --> `View Results Tree`
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Create View Results Tree.png">}}

View Results Tree界面
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/View Results Tree.png">}}

### Summary Report

在`Thread Group`右键 --> `Add` --> `Listener` --> `Summary Report`
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Create Summary Report.png">}}

Summary Report界面
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Summary Report.png">}}

最后需要将测试保存为*.jmx文件，这里将访问wikipedia的请求保存为wikipedia.jmx
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/jmx.png">}}

## 命令行执行测试
命令说明
```shell
jmeter -n -t [jmx file] -l [results file] -e -o [Path to web report folder]
```
具体执行命令
```shell
./jmeter -n -t ./wikipedia.jmx -l wikipedia/result/wikipedia.txt -e -o wikipedia
```
执行过程
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Command execute test.png">}}

## 测试结果
命令行执行测试完毕后，可以在`wikipedia/index.html`中查看执行的具体情况
{{< figure src="/images/blog/stress-testing-apis-using-jmeter/Test Result Web Page.png">}}


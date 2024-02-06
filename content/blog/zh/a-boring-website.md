---
title: 一个无聊的网站
date: 2023-09-14T16:47:12+08:00
tags: ["boring-website", "AWS"]
series: ["Boring website"]
featured: true
---

本文是无聊网站的第一篇，主要介绍了无聊网站[freeupdateme.com](https://freeupdateme.com)的建站起因、网站功能、网站架构、技术难点、代码模板和参考资料。

<!--more-->

## 建站起因

很久之前，在网上看到过一篇文章，讲述了一个无意义的网站。这个网站只有一个按钮，一个倒计时和一行文字。如果用户在一定的时间内（也没有具体说明是多久）不点击这个按钮，该网站将永远关闭，据说到发文为止，网站也不曾关闭。可惜，我从未找到这个网站。但是某一天当我又想起这个事情的时候，突然意识到，我可以自己去验证这个事情。

## 网站功能

### 基本功能

网站的功能其实很简单，主要是为了统计用户操作网页这个动作之间的时间间隔。用户需要操作一个输入框，输入并确认。然后将用户输入的内容显示在页面上。当用户再次输入时，显示内容会被覆盖。后台数据库会保存用户输入内容，输入时间和IP地址。这样就可以拿到用户输入的内容和时间，进行分析。

### 趣味功能
为了更好玩，我增加了下面的一些功能，并附上了网站截图：
- 添加了一个计时器，显示最后一次输入到当前的时间
- 添加了IP地址信息，利用[ip2region](https://github.com/lionsoul2014/ip2region)解析IP地址获得
- 添加了今日剩余更新次数，今日更新次数和总更新次数
- 添加了词云图，使用[结巴中文分词](https://github.com/fxsjy/jieba)，图片绘制使用[wordcloud2](https://wordcloud2-js.timdream.org/#love)在线生成。

{{< figure src="/images/blog/a-boring-website/website-screenshot.png" height="500">}}

## 网站架构
功能确定后，就需要确定网站架构。在这里我们使用AWS云，主要服务有Route53, API Gateway、Lambda、DynamoDB和S3，整体架构如下图。

{{< figure src="/images/blog/a-boring-website/boring-website-architecture.png">}}

网站响应流程：

1. 浏览器发送请求到Route53，S3（用于加载页面的图片资源）
2. Route53将请求路由到API Gateway
3. API Gateway触发Lambda
4. Lambda运行并处理逻辑


## 技术难点

在构建网站时遇到了一个问题，VPC内Lambda不能够访问Dynamodb。解决这个问题可以有三个思路
1. 给Lambda访问公网能力，这样通过公网连接Dynamodb。

- 这个方案在执行后还是不能够连接Dynamodb，原因是Lambda作为serverless不能够有public ip，参考stackoverflow的[回答](https://stackoverflow.com/questions/52992085/why-cant-an-aws-lambda-function-inside-a-public-subnet-in-a-vpc-connect-to-the)
2. 使用dynamoDB的VPC endpoint，Lambda不通过公网连接Dynamodb。

- 这个方案可行，但是需要创建额外资源，而且Lambda实际上是可以不部署在VPC内的。所以就有了第三个方案。
3. 将Lambda部署在非VPC环境内，这样Lambda和Dynamodb在同一个环境，就可以访问了

- 这个方案架构简单，仅部署需要的资源，不需要配置VPC和Subnet，省事了不少。

## 代码模板
在这里仅展示可复用的代码，AWS资源创建cloudformation和部署脚本
### 资源模板

{{< toggle summary="click here to show `resources.yml` file" >}}

```yaml
---
AWSTemplateFormatVersion: "2010-09-09"
Description: Free update me
Transform: "AWS::Serverless-2016-10-31"
Parameters:
  AppName:
    Type: String
Resources:
  API:
    Type: AWS::Serverless::Api
    Properties:
      StageName: api
      TracingEnabled: true
      OpenApiVersion: 3.0.2
  LambdaFunction:
    Type: "AWS::Serverless::Function"
    Properties:
      CodeUri: ../dist/
      FunctionName: !Ref AppName
      Environment:
        Variables:
          REGION: !Ref "AWS::Region"
      Handler: freeUpdateMeHandler.handler
      MemorySize: 128
      Runtime: nodejs18.x
      Timeout: 60
      Policies:
        - Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - dynamodb:PutItem
                - dynamodb:GetItem
              Resource: !GetAtt FreeUpdateMeDynamoDBTable.Arn
      Tracing: Active
      Events:
        getEndpoint:
          Type: Api
          Properties:
            RestApiId: !Ref API
            Path: /
            Method: GET
        postEndpoint:
          Type: Api
          Properties:
            RestApiId: !Ref API
            Path: /
            Method: POST
  LogGroup:
    Type: "AWS::Logs::LogGroup"
    DependsOn: LambdaFunction
    Properties:
      LogGroupName: !Join
        - /
        - - /aws/lambda
          - !Ref LambdaFunction
      RetentionInDays: 1
  FreeUpdateMeDynamoDBTable:
    Type: "AWS::DynamoDB::Table"
    Properties:
      TableName: free-update-me
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
```

{{</toggle>}}

### 脚本

````shell
aws cloudformation package \
  --region $AWS_REGION \
	--template-file $ORIGINAL_TEMPLATE_PATH \
	--output-template-file $COMPILED_TEMPLATE_PATH \
	--s3-bucket $BUCKET \
	--s3-prefix $APP_NAME \
	--use-json

aws cloudformation deploy \
    --region $AWS_REGION \
    --stack-name $APP_NAME \
    --template-file $COMPILED_TEMPLATE_PATH \
    --parameter-overrides file://$PARAMETER_FILE_PATH \
    --tags $TAGS \
    --capabilities CAPABILITY_IAM
````

## 参考资料
1. [what-is-the-downside-of-not-running-aws-lambda-functions-in-a-vpc](https://stackoverflow.com/questions/45580610/what-is-the-downside-of-not-running-aws-lambda-functions-in-a-vpc)
2. [why-cant-an-aws-lambda-function-inside-a-public-subnet-in-a-vpc-connect-to-the- internet](https://stackoverflow.com/questions/52992085/why-cant-an-aws-lambda-function-inside-a-public-subnet-in-a-vpc-connect-to-the)


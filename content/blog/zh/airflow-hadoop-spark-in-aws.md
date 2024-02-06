---
title: 在AWS上构建基于Airflow Hadoop和Spark大数据平台
date: 2024-01-11T20:00:00+08:00
tags: ["AWS", "Airflow", "Hadoop", "Spark", "Hive"]
series: ["大数据"]
featured: true
---

本文介绍了AWS云平台服务EMR Serverless，Airflow托管服务MWAA（Managed Workflows for Apache Airflow），构建了基于AWS云平台的大数据平台，最后以一个实例演示了大数据平台的工作流程。

<!--more-->

## 大数据平台
现如今是一个数据时代，我们每天都在产生数据，海量数据的分析离不开大数据技术的支持。大数据是一个很宽泛的概念，但是一个大数据平台总结起来一般有三部分，数据接入，数据处理和数据分析。

- **数据接入**：是将数据写入数据仓库中，AWS可以使用S3作为数据仓库存储数据。
- **数据处理**：是对数据做清洗和ETL转化操作，是大数据的核心部分，常见的框架是 Apache Hadoop和 Apache Spark，AWS中 EMR 服务可以托管Hadoop和Spark框架，简化其运行过程。
- **数据分析**：是大数据的应用部分，最直接的方法就是使用 Hive 以 SQL 的方式对数据进行查询分析，AWS中对应的服务是Athena。

本文主要着重于数据处理和分析部分，并且引入Airflow调度框架，构建了企业级的大数据处理分析平台。架构如下图：
{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/架构图.png">}}

- **[EMR Serverless](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/emr-serverless.html)**：Amazon EMR是一个托管集群平台，可简化在AWS上运行大数据框架（如 Apache Hadoop 和 Apache Spark）的过程，以处理和分析海量数据。
而 Amazon EMR Serverless 是 Amazon EMR 的一个部署选项，可提供无服务器运行时环境，类似于AWS Lambda和EC2的关系，Serverless可以按需收费，极大的降低了成本。

- **[MWAA](https://docs.aws.amazon.com/mwaa/latest/userguide/what-is-mwaa.html)**：Amazon Managed Workflows for Apache Airflow 是一项适用于 Apache Airflow 的托管式编排服务，能够在AWS云中大规模设置和操作数据管道。
- **[Glue](https://docs.aws.amazon.com/zh_cn/glue/latest/dg/what-is-glue.html)** 和 **[Athena](https://docs.aws.amazon.com/zh_cn/athena/latest/ug/what-is.html)**：可以理解为 Hive，AWS 在 Hive 上做了一些封装，使其用起来更方便。

## 构建数据仓库
### 模板文件
使用 Cloudformation 模板构建 MWAA 和 EMR Serverless。模板比较长，其中还创建了 S3 Bucket 用来存储数据，IAM Role 用来赋予 Spark Job 读取和写入数据权限，
以及VPC、Subnet、InternetGateway等网络资源，如果对于AWS基础网络资源感兴趣，可以查看这一篇博客 [AWS多可用区的VPC网络环境](https://hui61.com/blog/zh/aws-multiple-available-zones-vpc/)。

{{< toggle summary="点击显示代码 `mwaa_emr_serverless.yaml` file" >}}

```yaml
---
AWSTemplateFormatVersion: 2010-09-09
Resources:
  # S3 Bucket for logs
  EMRServerlessBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: mwaa-emr-serverless
      PublicAccessBlockConfiguration:
        BlockPublicAcls: True
        BlockPublicPolicy: True
        IgnorePublicAcls: True
        RestrictPublicBuckets: True
      VersioningConfiguration:
        Status: Enabled

  # IAM resources
  EMRServerlessJobRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - emr-serverless.amazonaws.com
            Action:
              - "sts:AssumeRole"
      Description: "Service role for EMR Studio"
      Policies:
        - PolicyName: GlueAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "glue:GetDatabase"
                  - "glue:DeleteTable"
                  - "glue:GetDataBases"
                  - "glue:CreateDatabase"
                  - "glue:CreateTable"
                  - "glue:GetTable"
                  - "glue:GetTables"
                  - "glue:GetPartition"
                  - "glue:GetPartitions"
                  - "glue:CreatePartition"
                  - "glue:BatchCreatePartition"
                  - "glue:GetUserDefinedFunctions"
                Resource: "*"
        - PolicyName: S3Access
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "s3:GetObject"
                  - "s3:ListBucket"
                Resource: "*"
              - Effect: Allow
                Action:
                  - "s3:PutObject"
                  - "s3:DeleteObject"
                Resource:
                  - !Sub "arn:aws:s3:::${EMRServerlessBucket}/*"

  SparkApplication:
    Type: AWS::EMRServerless::Application
    Properties:
      Name: spark-3.2
      ReleaseLabel: emr-6.6.0
      Type: Spark
      MaximumCapacity:
        Cpu: 200 vCPU
        Memory: 100 GB
      AutoStartConfiguration:
        Enabled: true
      AutoStopConfiguration:
        Enabled: true
        IdleTimeoutMinutes: 100
      InitialCapacity:
        - Key: Driver
          Value:
            WorkerCount: 3
            WorkerConfiguration:
              Cpu: 2 vCPU
              Memory: 4 GB
              Disk: 21 GB
        - Key: Executor
          Value:
            WorkerCount: 4
            WorkerConfiguration:
              Cpu: 1 vCPU
              Memory: 4 GB
              Disk: 20 GB

  MWAAEnvironment:
    Type: AWS::MWAA::Environment
    Properties:
      Name: !Join
        - "-"
        - - emr-serverless-demo
          - !Select [0, !Split [-, !Select [2, !Split [/, !Ref AWS::StackId]]]]
      EnvironmentClass: mw1.small
      AirflowVersion: 2.2.2
      NetworkConfiguration:
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
        SecurityGroupIds:
          - !Ref MWAASecurityGroupIngress
      DagS3Path: airflow/dags
      SourceBucketArn: !Sub "arn:aws:s3:::${EMRServerlessBucket}"
      ExecutionRoleArn: !GetAtt MWAAExecutionRole.Arn
      WebserverAccessMode: PUBLIC_ONLY
      LoggingConfiguration:
        WebserverLogs:
          Enabled: True
          LogLevel: INFO
        SchedulerLogs:
          Enabled: True
          LogLevel: INFO

  MWAASecurityGroupIngress:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Inbound access to MWAA"
      VpcId: !Ref VPC

  MWAASecurityGroupSelfAllow:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref MWAASecurityGroupIngress
      IpProtocol: "-1"
      SourceSecurityGroupId: !Ref MWAASecurityGroupIngress

  MWAAExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - airflow.amazonaws.com
                - airflow-env.amazonaws.com
            Action:
              - "sts:AssumeRole"
      Description: "Service role for MWAA"
      Policies:
        - PolicyName: MWAA-Execution-Policy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "s3:GetObject*"
                  - "s3:GetBucket*"
                  - "s3:List*"
                Resource:
                  - !Sub "arn:aws:s3:::${EMRServerlessBucket}"
                  - !Sub "arn:aws:s3:::${EMRServerlessBucket}/*"
              - Effect: Allow
                Action:
                  - "airflow:PublishMetrics"
                Resource: !Sub "arn:aws:airflow:${AWS::Region}:${AWS::AccountId}:environment/emr-serverless-demo"
              - Effect: Allow
                Action:
                  - "logs:CreateLogStream"
                  - "logs:CreateLogGroup"
                  - "logs:PutLogEvents"
                  - "logs:GetLogEvents"
                  - "logs:GetLogRecord"
                  - "logs:GetLogGroupFields"
                  - "logs:GetQueryResults"
                Resource: !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:airflow-emr-serverless-demo-*"
              - Effect: Allow
                Action:
                  - "logs:DescribeLogGroups"
                  - "cloudwatch:PutMetricData"
                Resource: "*"
              - Effect: Allow
                Action:
                  - "sqs:ChangeMessageVisibility"
                  - "sqs:DeleteMessage"
                  - "sqs:GetQueueAttributes"
                  - "sqs:GetQueueUrl"
                  - "sqs:ReceiveMessage"
                  - "sqs:SendMessage"
                Resource: !Sub "arn:aws:sqs:${AWS::Region}:*:airflow-celery-*"
              - Effect: Allow
                Action:
                  - "kms:Decrypt"
                  - "kms:DescribeKey"
                  - "kms:GenerateDataKey*"
                  - "kms:Encrypt"
                NotResource: !Sub "arn:aws:kms:*:${AWS::AccountId}:key/*"
                Condition:
                  StringLike:
                    "kms:ViaService": !Sub "sqs.${AWS::Region}.amazonaws.com"
        - PolicyName: AirflowEMRServerlessExecutionPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "emr-serverless:CreateApplication"
                  - "emr-serverless:GetApplication"
                  - "emr-serverless:StartApplication"
                  - "emr-serverless:StopApplication"
                  - "emr-serverless:DeleteApplication"
                  - "emr-serverless:StartJobRun"
                  - "emr-serverless:GetJobRun"
                Resource: "*"
              - Effect: Allow
                Action:
                  - "iam:PassRole"
                Resource:
                  - !GetAtt EMRServerlessJobRole.Arn
                Condition:
                  StringLike:
                    "iam:PassedToService": "emr-serverless.amazonaws.com"

  # Network resources
  VPC:
    Properties:
      # Default CIDR block for public subnet
      CidrBlock: 172.31.0.0/16
      EnableDnsHostnames: "true"
      Tags:
        - Key: for-use-with-amazon-emr-managed-policies
          Value: true
    Type: AWS::EC2::VPC

  VPCDHCPAssociation:
    Properties:
      DhcpOptionsId: { Ref: VPCDHCPOptions }
      VpcId: { Ref: VPC }
    Type: AWS::EC2::VPCDHCPOptionsAssociation

  VPCDHCPOptions:
    Properties:
      DomainName: "${AWS::Region}.compute.internal"
      DomainNameServers: [AmazonProvidedDNS]
    Type: AWS::EC2::DHCPOptions

  # CIDR block for private subnets
  VpcCidrBlock1:
    Type: AWS::EC2::VPCCidrBlock
    Properties:
      VpcId: { Ref: VPC }
      CidrBlock: 172.16.0.0/16

  GatewayAttachment:
    Properties:
      InternetGatewayId: { Ref: InternetGateway }
      VpcId: { Ref: VPC }
    Type: AWS::EC2::VPCGatewayAttachment
  InternetGateway: { Type: "AWS::EC2::InternetGateway" }
  PublicRouteTableIGWRoute:
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: { Ref: InternetGateway }
      RouteTableId: { Ref: PublicRouteTable }
    Type: AWS::EC2::Route
  PublicRouteTable:
    Properties:
      Tags:
        - Key: Name
          Value: Public Route Table
      VpcId: { Ref: VPC }
    Type: AWS::EC2::RouteTable
  PublicSubnetRouteTableAssociation:
    Properties:
      RouteTableId: { Ref: PublicRouteTable }
      SubnetId: { Ref: PublicSubnet1 }
    Type: AWS::EC2::SubnetRouteTableAssociation
  PublicSubnet1:
    DependsOn: VpcCidrBlock1
    Properties:
      Tags:
        - Key: Name
          Value: PublicSubnet1
        - Key: for-use-with-amazon-emr-managed-policies
          Value: true
      VpcId: { Ref: VPC }
      MapPublicIpOnLaunch: "true"
      AvailabilityZone:
        Fn::Select:
          - 0
          - Fn::GetAZs: { Ref: "AWS::Region" }
      CidrBlock: 172.16.0.0/20
    Type: AWS::EC2::Subnet
  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIPAddress.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: NAT
  ElasticIPAddress:
    Type: AWS::EC2::EIP
    Properties:
      Domain: VPC
  # private subnets
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      Tags:
        - Key: Name
          Value: Private Route Table
      VpcId: { Ref: VPC }
  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: { Ref: PrivateRouteTable }
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: { Ref: NATGateway }
  PrivateSubnet1:
    DependsOn: VpcCidrBlock1
    Type: AWS::EC2::Subnet
    Properties:
      Tags:
        - Key: Name
          Value: PrivateSubnet1
        - Key: for-use-with-amazon-emr-managed-policies
          Value: true
      VpcId: { Ref: VPC }
      MapPublicIpOnLaunch: "false"
      AvailabilityZone:
        Fn::Select:
          - 0
          - Fn::GetAZs: { Ref: "AWS::Region" }
      CidrBlock: 172.31.0.0/20
  PrivateSubnetRouteTableAssociation1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: { Ref: PrivateRouteTable }
      SubnetId: { Ref: PrivateSubnet1 }
  PrivateSubnet2:
    DependsOn: VpcCidrBlock1
    Type: AWS::EC2::Subnet
    Properties:
      Tags:
        - Key: Name
          Value: PrivateSubnet2
        - Key: for-use-with-amazon-emr-managed-policies
          Value: true
      VpcId: { Ref: VPC }
      MapPublicIpOnLaunch: "false"
      AvailabilityZone:
        Fn::Select:
          - 1
          - Fn::GetAZs: { Ref: "AWS::Region" }
      CidrBlock: 172.31.16.0/20
  PrivateSubnetRouteTableAssociation2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: { Ref: PrivateRouteTable }
      SubnetId: { Ref: PrivateSubnet2 }
```

{{</toggle>}}

模板准备好了之后，使用下面的 shell 脚本执行，注意替换模板路径，其中创建 MWAA 需要较长时间。

```shell
aws cloudformation deploy \
    --stack-name "mwaa-emr-serverless" \
    --template-file "./src/aws/mwaa_emr_serverless.yaml" \
    --capabilities CAPABILITY_IAM
```

### 配置 Airflow
Airflow 创建完成后，还需要对其进行配置，点击 Edit 按钮进行配置，如下图：

{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/Airflow配置.png">}}

配置 Airflow 配置运行依赖

{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/配置Airflow运行依赖.png">}}

requirements.txt文件中依赖如下，需要将改文件上传至 S3 进行配置，配置完成后点击 Next 并 Save 使其生效。
```text
apache-airflow-providers-amazon==6.0.0
boto3>=1.23.9
```

## 实例演示
环境配置好之后，我们就可以用 Airflow 提交 Spark Job 到 EMR 了。按照下面步骤分别配置 Airflow 和 EMR 的任务。
### Airflow Dag
首先是准备 Airflow Dag 文件，将文件上传至`s3://mwaa-emr-serverless/airflow/dags/`

```python
from datetime import datetime

from airflow import DAG
from airflow.models import Variable
from airflow.providers.amazon.aws.operators.emr import EmrServerlessStartJobOperator

APPLICATION_ID = Variable.get("emr_serverless_application_id")
JOB_ROLE_ARN = Variable.get("emr_serverless_job_role")
S3_LOGS_BUCKET = Variable.get("emr_serverless_log_bucket")

with DAG(
        dag_id='example_emr_serverless_job',
        schedule_interval=None,
        start_date=datetime(2021, 1, 1),
        tags=['example'],
        catchup=False,
) as dag:
    job_starter = EmrServerlessStartJobOperator(
        task_id="start_job",
        application_id=APPLICATION_ID,
        execution_role_arn=JOB_ROLE_ARN,
        job_driver={
            "sparkSubmit": {
                "entryPoint": "s3://mwaa-emr-serverless/scripts/person.py",
                "entryPointArguments": ["s3://mwaa-emr-serverless/output/"]
            }
        },
        configuration_overrides={
            "monitoringConfiguration": {
                "s3MonitoringConfiguration": {
                    "logUri": f"s3://{S3_LOGS_BUCKET}/logs/"
                }
            },
        },
        config={"name": "sample-job"}
    )
```
这个 Dag 十分简单，使用 EmrServerlessStartJobOperator 提交一个 spark 任务，其中 entryPoint 和 entryPointArguments 是任务路径和参数。

Airflow Dag中用到了三个 Variable，
- **emr_serverless_application_id**: 为 EMR Application ID 需要在AWS Console 中 EMR Studio 界面找到，本例为：`00fg6k1re8jj6525` 
- **emr_serverless_job_role**: 为 EMRServerlessJobRole arn 我们在 cloudformation 中创建了，可以在AWS console中找到，本例为：`arn:aws:iam::320123455241:role/mwaa-emr-serverless-EMRServerlessJobRole-hGGPdYlKx3jF`
- **emr_serverless_log_bucket**: 为 Bucket Name，本例为：`mwaa-emr-serverless`

需要对其进行配置，如下图创建 Airflow Variable
{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/Airflow Variable配置.png">}}

### Spark Job Script
其次准备 Spark 任务的脚本，将脚本上传至 `s3://mwaa-emr-serverless/scripts/person.py`
```python
import os
import sys

from pyspark.sql import SparkSession
from pyspark.sql.functions import lower

if __name__ == "__main__":
    spark = (SparkSession
             .builder
             .appName("person")
             .config("spark.hadoop.hive.metastore.client.factory.class",
                     "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory")
             .config("spark.hadoop.hive.metastore.glue.catalogid", "320123455241")
             .config("spark.hadoop.fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
             .enableHiveSupport()
             .getOrCreate())

    output_path = None

    if len(sys.argv) > 1:
        output_path = sys.argv[1]
    else:
        print("S3 output location not specified printing top 10 results to output stream")

    region = os.getenv("AWS_REGION")
    reader = spark.read.format("json")

    df = reader.load("s3://mwaa-emr-serverless/input/")
    lower_first_name_df = df.withColumn("first_name", lower(df["first_name"]))
    lower_last_name_df = lower_first_name_df.withColumn("last_name", lower(lower_first_name_df["last_name"]))
    # 创建 Glue database
    spark.sql("create database if not exists test_database")

    if output_path:
        lower_last_name_df.write \
            .option("path", output_path) \
            .mode("overwrite") \
            .format("parquet") \
            .saveAsTable("test_database.person")

        print("person job completed successfully. Refer output at S3 path: " + output_path)
    else:
        print("person job completed successfully.")

    spark.stop()
```
这段 spark job 是从 `s3://mwaa-emr-serverless/input/` 读取数据，然后将数据中 first_name 和 last_name 转为小写。所以我们还要准备需要处理的数据。
将下面3个json文件保存在 `s3://mwaa-emr-serverless/input/`
```text
// 4.json
{"first_name": "Louis", "last_name": "Hui", "age": 31, "describe": "A programmer"}

// 5.json
{"first_name": "Jack", "last_name": "Zhang", "age": 30, "describe": "A engineer"}

// 6.json
{"first_name": "Mike", "last_name": "Luo", "age": 31, "describe": "A singer"}
{"first_name": "Marry", "last_name": "Zhan", "age": 19, "describe": "A dancer"}
```

### 结果展示
触发 Airflow Dag 后会提交 Spark Job，执行会有一段时间，因为 EMR Serverless 是按需收费，所以启动需要时间，可以在AWS Console中查看任务执行状态
{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/EMR Job Successfully.png">}}

执行完毕后，我们就可以在 S3 Bucket 里面查看处理后的数据
{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/S3 Output.png">}}

并且可以在 Athena 中使用 SQL 查询数据
{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/Athena Query Result.png">}}

值得一提的是本次 Demo 的花费，得益于 AWS EMR Serverless 的低成本，本次演示的花费仅为0.39$，MWAA 的花费是根据时长收费的，所以不使用的时候最好删掉。
{{< figure src="/images/blog/airflow-hadoop-spark-in-aws/花费.png">}}

## 参考资料
1. https://docs.aws.amazon.com/zh_cn/emr/latest/ManagementGuide/emr-what-is-emr.html
2. https://docs.aws.amazon.com/zh_cn/mwaa/latest/userguide/configuring-dag-folder.html#configuring-dag-folder-mwaaconsole
3. https://docs.aws.amazon.com/mwaa/latest/userguide/what-is-mwaa.html
4. https://docs.aws.amazon.com/zh_cn/glue/latest/dg/what-is-glue.html
5. https://docs.aws.amazon.com/zh_cn/athena/latest/ug/what-is.html


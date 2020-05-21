+++
title = "Serverless Lambda Dynamodb API"
date = "2020-02-12"
description = "a reusable terraform module to spin up an api in 5 mins"
aliases = ["Post","post","blog","blog-page"]
tags = ["python3", "serverless", "lambda", "dynamodb", "api", "2019nCov", "aws-api-gateway",]
+++

This post is about how to terraform up a serverless, scalable and resilient lambda API. I am using the [2019nCov-api](https://applegreengrape.github.io/posts/2019ncov-api/) as an example. In addition, we will have a look at terraform's built-in functions to make a more reusable modules for api gateway and lambdas. 

{{< image src="https://media.giphy.com/media/3o7ZeqkYTvaL3lGjCw/giphy.gif" position="center" style="width: 25%" >}}

#### Use Case 

The use case is I want to build a simple API to allow user to retrieve [coronavirus data](/posts/2019ncov-api/). There are two major tables. One is [DXY stats](https://ncov.dxy.cn/ncovh5/view/pneumonia), another one is people's [travel data](http://2019ncov.nosugartech.com/). And I want to allow people to query specific fields via url query parameters. Also I want to schedule the coronavirus data injection daily to dynamodb.

#### Architecture Diagram 

{{< image src="https://raw.githubusercontent.com/applegreengrape/2019nCov-API/master/img/serverless-backend-api-diagram.png" position="center" style="width: 125%" >}}

👉 [click here for the diagram](https://raw.githubusercontent.com/applegreengrape/2019nCov-API/master/img/serverless-backend-api-diagram.png)

Okie dokie, now let's look at the tools we can use from AWS. 
- AWS API Gateway 
- AWS Cloudwatch Event
- AWS Lambda 
- AWS Dynamodb

#### Let's terraform it up

As we can see from the diagram, the difficult part is the AWS API Gateway . We will need one API Gateway with two pathes, each path is using `GET` methods. Let's use terraform's functions to make a more reusable api gateway module. To create an AWS API gateway, you will need:

- `aws_api_gateway_rest_api`
- `aws_api_gateway_resource`
- `aws_api_gateway_method`
- `aws_api_gateway_integration`
- `aws_api_gateway_integration_response`
- `aws_api_gateway_deployment`

Okie, dokie Let's look at the code. The key to make this more reusable is to pair the endpoints.

{{< gist applegreengrape 22097bf6e2e3b4d0c155590aa84ececf >}}

Here is the output of the pairing 👇:

```bash
(base) pingzhous-MBP:lambda-tf-v12 pingzhouliu$ terraform state list module.api-gw
module.api-gw.aws_api_gateway_deployment.api-gw-module
module.api-gw.aws_api_gateway_integration.api-gw-module["GET dxy"]
module.api-gw.aws_api_gateway_integration.api-gw-module["GET travel"]
module.api-gw.aws_api_gateway_integration_response.api-gw-module["GET dxy 200"]
module.api-gw.aws_api_gateway_integration_response.api-gw-module["GET travel 200"]
module.api-gw.aws_api_gateway_method.api-gw-module["GET dxy"]
module.api-gw.aws_api_gateway_method.api-gw-module["GET travel"]
module.api-gw.aws_api_gateway_resource.api-gw-module["dxy"]
module.api-gw.aws_api_gateway_resource.api-gw-module["travel"]
module.api-gw.aws_api_gateway_rest_api.api-gw-module
module.api-gw.aws_lambda_permission.apigw_lambda["GET dxy"]
module.api-gw.aws_lambda_permission.apigw_lambda["GET travel"]
```

You can find the module's `main.tf` and `variables.tf` here 👇

{{< gist applegreengrape 4388e4507d788f162156aa41a7df7656 >}}

Now let's call the api-gw module:

```bash
module "api-gw" {
  source = "./api-gw"

  api-gw-name = "2019-nCov-API-GW"
  stage_name = "v1"
  endpoints = [
    {
      path = "dxy"
      method = "GET"
      lambda = "dxy"
      request_parameters = { 
        "method.request.querystring.date" = true
        "method.request.querystring.country" = true
        "method.request.querystring.cityName" = true
        "method.request.querystring.provinceName" = true
        "method.request.querystring.all" = true
        }
    },
    {
      path = "travel"
      method = "GET"
      lambda = "travel"
      request_parameters = { 
        "method.request.querystring.date" = true
        "method.request.querystring.start" = true
        "method.request.querystring.stop" = true
        "method.request.querystring.all" = true
        }
    }
  ]
}
```
🥳 🥳 🥳 Here you go! Your API is up 🤗 🤗 🤗
```bash
pingzhous-MBP:lambda-tf-v12 pingzhouliu$ terraform output
endpoint_id = https://4mmhkv7z9e.execute-api.eu-west-1.amazonaws.com/v1/
execution_arn = arn:aws:execute-api:eu-west-1:900665556514:4mmhkv7z9e

$ curl -s https://4mmhkv7z9e.execute-api.eu-west-1.amazonaws.com/v1/dxy?all=yes | jq -r .[].date | sort | uniq
2020-02-08
2020-02-09
2020-02-10
2020-02-11
2020-02-12

$ curl -s https://4mmhkv7z9e.execute-api.eu-west-1.amazonaws.com/v1/dxy?date=2020-02-12 | jq .[78]
{
  "id": "3275",
  "date": "2020-02-12",
  "country": "中国",
  "provinceName": "重庆市",
  "cityName": "忠县",
  "confirmedCount": "19",
  "suspectedCount": "NULL",
  "curedCount": "6",
  "deadCount": "NULL",
  "msg": " No Man is an Island  🏝  没有人是一座孤岛 @pingzhou| 平舟 ⛵"
}
```
👉 [click here to see more examples of this API](/posts/2019ncov-api/)

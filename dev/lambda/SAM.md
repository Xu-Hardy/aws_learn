---
title: 本地使用SAM模拟Lambda
categories: lambda
tags: 
    - Severless
    - SAM
---
AWS 提供了一个工具叫做 `AWS SAM (Serverless Application Model)`。以下是如何使用 SAM 在本地模拟 Lambda 函数的简单步骤：

1. **安装 AWS SAM CLI**

   需要先安装 [Docker](https://www.docker.com/get-started) 和 AWS SAM CLI。安装 Docker 之后，可以使用以下命令安装 SAM CLI：

   ```bash
   pip install aws-sam-cli
   ```

2. **初始化一个新的 SAM 项目**

   如果你还没有一个 SAM 项目，你可以使用以下命令来创建一个：

   ```bash
   sam init --runtime python3.8
   ```

   这将会创建一个新的 SAM 项目，并使用 Python 3.8 作为运行时。你可以选择你喜欢的其他运行时，如 nodejs, java 等。

3. **本地模拟 Lambda 函数**

   使用以下命令来启动本地的 Lambda 模拟器：

   ```bash
   sam local invoke
   ```

   这会启动一个 Docker 容器，模拟 Lambda 环境，并运行你的函数。

   如果你有一个 API Gateway 的定义，并希望模拟整个 API，你可以使用以下命令：

   ```bash
   sam local start-api
   ```

   然后你可以向 `http://127.0.0.1:3000` 发送请求来模拟 API 调用。

4. **传递事件数据**

   你可以使用 `-e` 参数传递一个事件文件给 `sam local invoke`。例如：

   ```bash
   sam local invoke -e events/event.json
   ```

   其中 `events/event.json` 是包含模拟事件数据的文件。

5. **查看日志**

   在模拟 Lambda 时，所有的输出和日志将直接打印到控制台，这使得调试和追踪问题变得更容易。

6. **停止模拟**

   使用 `Ctrl + C` 可以停止模拟器。
   
`AWS SAM (Serverless Application Model)` 是一个用于构建 Serverless 应用的工具。以下是一些 `SAM CLI` 的常用命令：

1. **初始化一个新的 SAM 项目**

   ```bash
   sam init
   ```

   使用上述命令可以初始化一个新的 SAM 项目。你可以选择一个模板、运行时或其他选项来开始。

2. **在本地模拟 Lambda 函数**

   ```bash
   sam local invoke [FunctionLogicalID] 
   ```

   使用该命令模拟 Lambda 函数执行。如果你的 template 文件中有多个函数，需要提供特定的 `FunctionLogicalID`。

3. **启动本地 API Gateway**

   ```bash
   sam local start-api
   ```

   如果你的 SAM 模板定义了 API Gateway，你可以使用这个命令在本地启动一个模拟的 API Gateway。

4. **生成模拟事件**

   ```bash
   sam local generate-event
   ```

   你可以使用这个命令来生成一个模拟的事件，如 S3 事件、API Gateway 事件等。

5. **打包 SAM 应用**

   ```bash
   sam package --template-file template.yaml --output-template-file packaged.yaml --s3-bucket YOUR_S3_BUCKET
   ```

   使用上述命令将你的应用打包并上传到 S3 存储桶，为部署做准备。

6. **部署 SAM 应用**

   ```bash
   sam deploy --template-file packaged.yaml --stack-name YOUR_STACK_NAME --capabilities CAPABILITY_IAM
   ```

   使用这个命令部署你的 SAM 应用到 AWS。

7. **快速打包和部署**

   对于打包和部署，你也可以直接使用以下命令，SAM 会自动进行打包和部署：

   ```bash
   sam deploy --guided
   ```

8. **验证 SAM 模板**

   ```bash
   sam validate
   ```

   使用这个命令验证你的 `template.yaml` 文件的语法和结构。

9. **构建 SAM 应用**

   ```bash
   sam build
   ```

   如果你的函数依赖外部库或需要编译，使用这个命令可以构建它们。

10. **查看 SAM 日志**

   ```bash
   sam logs -n FunctionLogicalID --stack-name YOUR_STACK_NAME
   ```

   使用这个命令可以查看部署在 AWS 上的函数的日志。

这只是 SAM CLI 的常用命令的概述。为了充分利用它，建议查阅 [AWS SAM CLI 官方文档](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-command-reference.html) 以了解更多详情和选项。

注意：虽然 SAM 提供了一个非常接近真实 AWS 环境的模拟，但仍然可能存在一些差异。因此，在部署到真实的 AWS 环境之前，最好还是进行充分的测试。
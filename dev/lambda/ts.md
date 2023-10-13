---
title: 在lambda上运行 Typescript
categories: lambda
tags: 
    - Typescript
---
要在AWS Lambda上运行TypeScript，你需要将TypeScript代码编译为JavaScript，然后将其部署到Lambda。以下是如何实现这个目标的简要步骤：

1. 安装TypeScript：

在你的项目根目录下运行以下命令，安装TypeScript和类型定义：

```bash
npm init -y
npm install typescript --save-dev
npm install @types/node --save-dev
```

2. 配置TypeScript：

在项目根目录下创建一个名为`tsconfig.json`的文件，添加以下内容：

```json
{ 
  "compilerOptions": {
    "target": "es2018",
    "module": "commonjs",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src"]
}
```

这会将`src`目录下的TypeScript代码编译到`dist`目录下的JavaScript代码。

3. 编写Lambda函数：

在`src`目录下创建一个名为`handler.ts`的文件，并编写你的Lambda函数：

```typescript
import { APIGatewayProxyHandler } from 'aws-lambda';

export const hello: APIGatewayProxyHandler = async (event, context) => {
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Hello from TypeScript Lambda!',
    }),
  };
};
```

4. 编译TypeScript代码：

在项目根目录下运行以下命令，将TypeScript代码编译为JavaScript：

```bash
npx tsc
```

编译后的JavaScript代码将出现在`dist`目录下。

5. 部署Lambda函数：

你可以使用AWS CLI、Serverless Framework、AWS CDK等工具将编译后的Lambda函数部署到AWS。在这里，我们使用AWS CLI。

首先，安装并配置[AWS CLI](https://aws.amazon.com/cli/)。

然后，将编译后的代码压缩为一个ZIP文件：

```bash
cd dist
zip -r lambda.zip .
```

接下来，使用AWS CLI创建一个Lambda函数：

```bash
aws lambda create-function \
  --function-name your-function-name \
  --runtime nodejs14.x \
  --handler handler.hello \
  --role arn:aws:iam::<your-account-id>:role/service-role/your-lambda-role \
  --zip-file fileb://lambda.zip
```

将`your-function-name`替换为你希望使用的函数名称，将`<your-account-id>`替换为你的AWS帐户ID，将`your-lambda-role`替换为适当的IAM角色。

完成上述步骤后，你的TypeScript Lambda函数将部署到AWS Lambda，并准备好运行。你可以使用`aws lambda invoke`命令或AWS管理控制台测试和调用你的函数。
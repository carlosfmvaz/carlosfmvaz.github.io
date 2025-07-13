---
title: "Building a Modern Serverless Workflow: Node.js, TypeScript, Serverless Framework, and AWS Lambda with Terraform"
description: >-
    Hey folks, it's time to dive into something practical and super useful! 🚀<br><br>

    Today I’ll walk you through how to deploy a Node.js Lambda function to AWS using
    the Serverless Framework and Terraform. This setup is perfect if you want a scalable
    and modern serverless application that plays nicely with infrastructure-as-code.

author: carlos
date: 2025-07-13
categories: [AWS, Serverless]
tags: [node.js, lambda, terraform, serverless, typescript]
pin: true
image: /assets/img/nodejs-lambda.png
---

## Project Overview

This project is a template for developing AWS Lambda functions using modern JavaScript tooling. It leverages:

- **Node.js** for the runtime environment.
- **TypeScript** for type safety and maintainability.
- **Serverless Framework** to define and organize Lambda functions and their event sources.
- **Terraform** for infrastructure-as-code, handling the provisioning of AWS resources.
- **Docker** for local development and environment parity.
- **Jest** for unit testing.

The codebase is organized to separate application logic (`src/`), tests (`tests/`), and infrastructure (`terraform/`). Supporting files like `serverless.yml`, `tsconfig.json`, and Docker-related configs enable a smooth developer experience.

---

## How It Works

### 1. Application Layer

All business logic and Lambda handlers are written in TypeScript under the `src/` directory. TypeScript is compiled to JavaScript before deployment, ensuring compatibility with AWS Lambda’s Node.js runtime.

Here’s the actual Lambda handler from this project:

```typescript
import { APIGatewayProxyEvent, Context } from "aws-lambda";
import ExampleController from "./example-controller";

export const handler = async (event: APIGatewayProxyEvent | any, context: Context | any): Promise<any> => {
    try {
        const { httpMethod } = event;
        const action = event.pathParameters.proxy.includes('/') ? event.pathParameters.proxy.split('/')[0] : event.pathParameters.proxy;
        if (!methods[httpMethod.toLowerCase()] || !methods[httpMethod.toLowerCase()][action]) {
            return {
                statusCode: 404,
                body: JSON.stringify({ message: 'Not found' }),
            };
        }
        const response = await methods[httpMethod.toLowerCase()][action](event, context);
        return response;
    } catch (error) {
        return {
            statusCode: 500,
            body: JSON.stringify({ message: 'An error occurred' }),
        };
    }
};

const example_controller = new ExampleController();

const methods: any = {
    post: {
        hello: async (event: APIGatewayProxyEvent, context: Context) => await example_controller.helloWorld(event, context),
    },
    get: {
        hello: async (event: APIGatewayProxyEvent, context: Context) => await example_controller.helloWorld(event, context),
    },
    put: {
        hello: async (event: APIGatewayProxyEvent, context: Context) => await example_controller.helloWorld(event, context),
    },
    delete: {
        hello: async (event: APIGatewayProxyEvent, context: Context) => await example_controller.helloWorld(event, context),
    }
}
```

The `serverless.yml` file describes the Lambda functions, their handlers, and event triggers. Here’s the actual configuration:

```yaml
service: lambda-example

provider:
  name: aws
  runtime: nodejs18.x
  stage: dev
  region: us-east-1

plugins:
  - serverless-plugin-typescript
  - serverless-offline

package:
  individually: false

functions:
  hello:
    handler: src/index.handler
    events:
      - http:
          path: /
          method: get
      - http:
          path: /
          method: post
      - http:
          path: /
          method: put
      - http:
          path: /
          method: delete
      - http:
          path: /{proxy+}
          method: any

custom:
  serverless-offline:
    httpPort: 3000
    host: 0.0.0.0
    reloadHandler: true
```

This abstraction allows developers to focus on code, while the framework handles packaging and wiring up AWS resources.

### 2. Local Development

Local development is streamlined with several tools:

- **Serverless Offline**: Emulates AWS Lambda and API Gateway locally, allowing you to invoke functions and test endpoints without deploying to AWS.
- **Docker**: The included `Dockerfile.dev` and `docker-compose.yml` files let you spin up a development container that mirrors the Lambda environment.

Here’s the actual Dockerfile used for development:

```dockerfile
FROM node:22-alpine
COPY . /app
WORKDIR /app
RUN npm install
```

- **Jest**: Unit tests live in the `tests/` directory and can be run with `npm test`, ensuring code quality before deployment.

The TypeScript configuration is managed in `tsconfig.json`:

```json
{
    "compilerOptions": {
        "target": "es2016",
        "lib": [
            "es2016"
        ],
        "module": "commonjs",
        "rootDir": "src",
        "allowJs": true,
        "outDir": ".build/src",
        "esModuleInterop": true,
        "forceConsistentCasingInFileNames": true,
        "strict": true,
        "noImplicitAny": true,
        "skipLibCheck": true,
        "resolveJsonModule": true,
        "moduleResolution": "node",
        "types": [
            "node",
            "jest"
        ]
    },
    "ts-node": {
        "compilerOptions": {
            "module": "ESNext",
            "resolveJsonModule": true
        }
    },
    "ts-jest": {
        "diagnostics": {
            "warnOnly": true
        }
    },
    "exclude": [
        "tests",
        "node_modules",
        "build",
        "**/*.js"
    ]
}
```

### 3. Infrastructure as Code

While the Serverless Framework can deploy resources directly, this project uses **Terraform** for infrastructure management. All AWS resources (including the Lambda function, IAM roles, and API Gateway) are defined in HCL under `terraform/`.

Here’s the actual Lambda resource definition:

```hcl
data "aws_iam_policy_document" "assume_role" {
  statement {
    actions = [
      "sts:AssumeRole"
    ]
    effect = "Allow"
    principals {
        type        = "Service"
        identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_role" {
  name               = "lambda_role"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/../dist/"
  output_path = "${path.module}/../lambda.zip"
}

resource "aws_lambda_function" "example_lambda" {
  function_name = "example_lambda"
  role          = aws_iam_role.lambda_role.arn
  handler       = "src/index.handler"
  runtime       = "nodejs18.x"
  filename      = data.archive_file.lambda_zip.output_path

  environment {
    variables = {
      EXAMPLE_VAR = "example_value"
    }
  }

  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
}
```

This approach provides greater control and visibility over infrastructure, and makes it easy to manage resources outside the scope of Serverless.

### 4. Deployment Workflow

A sample `build-lambda.sh` script is included to automate the build and packaging process. It compiles TypeScript, bundles the code, and prepares a ZIP file for deployment.

Here’s the actual script:

```bash
#!/bin/bash

# This script builds the AWS Lambda function for the Node.js example project.
npm run build
mkdir -p ./dist
cp package.json ./dist/package.json
cd ./dist
npm install --omit=dev
cp -r ../.build/* ./
```

**Important:** This script is for demonstration only. In production, you should use a CI/CD pipeline (such as GitHub Actions, GitLab CI, or AWS CodePipeline) to automate builds, tests, and deployments securely and reliably.

Deployment is handled by Terraform:

```bash
cd terraform
terraform init
terraform apply
```

Terraform provisions the Lambda function and any associated AWS resources.

---

## Local Configuration

To get started locally:

- Install dependencies with `npm install`.
- Use `npx serverless offline` to emulate AWS Lambda and test endpoints.
- Optionally, use Docker Compose to run the development environment in a container.
- Run tests with `npm test`.

The project is designed to be developer-friendly, with clear separation of concerns and support for modern tooling.

---

## Why This Stack?

- **TypeScript** improves code quality and maintainability.
- **Serverless Framework** abstracts away much of the AWS boilerplate, letting you focus on business logic.
- **Terraform** provides a single source of truth for infrastructure, making it easy to manage, version, and review changes.
- **Docker** ensures consistency between local and production environments.

---

## Lessons Learned

- Combining Serverless Framework and Terraform gives you the best of both worlds: rapid development and robust infrastructure management.
- Investing in local emulation (via Serverless Offline and Docker) pays off in faster feedback loops and fewer surprises in production.
- Automating deployment with CI/CD is essential for real-world projects; scripts are useful for illustration, but pipelines are the standard.

---

## Conclusion

This project serves as a blueprint for building, testing, and deploying serverless applications with modern tools. By combining Node.js, TypeScript, Serverless Framework, and Terraform, you can create scalable, maintainable, and production-ready AWS Lambda functions with confidence.

Feel free to explore the codebase, adapt it for your own needs, and integrate these patterns into your workflow!

[Check the Repository!](https://github.com/carlosfmvaz/aws-playground/tree/master/lambda)
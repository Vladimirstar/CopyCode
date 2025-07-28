
下面是一个完整的Java Maven项目，演示了使用AWS API Gateway、Lambda、Step Functions链式调用和EventBridge的集成，并通过Terraform部署。

### 项目结构
```
aws-demo/
├── src/
│   ├── main/
│   │   ├── java/com/example/
│   │   │   ├── StartExecutionHandler.java     # API Gateway触发的Lambda
│   │   │   ├── StepFunctionTask1Handler.java  # Step Functions任务1
│   │   │   ├── StepFunctionTask2Handler.java  # Step Functions任务2
│   │   │   └── EventBridgeHandler.java        # EventBridge触发的Lambda
│   │   └── resources/
│   │       └── statemachine.json              # Step Functions定义
│   └── test/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── pom.xml
```

### 1. Java代码实现

#### `StartExecutionHandler.java` (API Gateway -> Lambda)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sfn.SfnClient;
import software.amazon.awssdk.services.sfn.model.StartExecutionRequest;

import java.util.HashMap;
import java.util.Map;

public class StartExecutionHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    // 从环境变量获取状态机ARN
    private static final String STATE_MACHINE_ARN = System.getenv("STATE_MACHINE_ARN");
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        try {
            SfnClient sfn = SfnClient.builder()
                .region(Region.of(System.getenv("AWS_REGION")))
                .build();

            // 启动Step Functions执行
            StartExecutionRequest request = StartExecutionRequest.builder()
                .stateMachineArn(STATE_MACHINE_ARN)
                .input(input.getBody())
                .build();
            
            String executionArn = sfn.startExecution(request).executionArn();
            
            // 返回响应
            Map<String, String> responseBody = new HashMap<>();
            responseBody.put("executionArn", executionArn);
            responseBody.put("message", "Step Functions started");
            
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(200)
                .withBody(responseBody.toString());
        } catch (Exception e) {
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(500)
                .withBody("{\"error\":\"" + e.getMessage() + "\"}");
        }
    }
}
```

#### `StepFunctionTask1Handler.java` (Step Functions任务1)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import java.util.Map;

public class StepFunctionTask1Handler implements RequestHandler<Map<String, Object>, Map<String, Object>> {

    @Override
    public Map<String, Object> handleRequest(Map<String, Object> input, Context context) {
        // 处理业务逻辑
        input.put("task1Result", "Processed by Task1");
        input.put("status", "SUCCESS");
        return input;
    }
}
```

#### `StepFunctionTask2Handler.java` (Step Functions任务2)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import java.util.Map;

public class StepFunctionTask2Handler implements RequestHandler<Map<String, Object>, Map<String, Object>> {

    @Override
    public Map<String, Object> handleRequest(Map<String, Object> input, Context context) {
        // 处理业务逻辑
        input.put("task2Result", "Processed by Task2");
        input.put("status", "COMPLETED");
        return input;
    }
}
```

#### `EventBridgeHandler.java` (EventBridge触发的Lambda)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.ScheduledEvent;

public class EventBridgeHandler implements RequestHandler<ScheduledEvent, String> {

    @Override
    public String handleRequest(ScheduledEvent event, Context context) {
        // 处理EventBridge事件
        System.out.println("Received EventBridge event: " + event.toString());
        return "Event processed at " + event.getTime();
    }
}
```

#### `statemachine.json` (Step Functions定义)
```json
{
  "Comment": "Chain Step Functions tasks",
  "StartAt": "Task1",
  "States": {
    "Task1": {
      "Type": "Task",
      "Resource": "${task1_lambda_arn}",
      "Next": "Task2"
    },
    "Task2": {
      "Type": "Task",
      "Resource": "${task2_lambda_arn}",
      "End": true
    }
  }
}
```

### 2. Maven配置 (`pom.xml`)
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>aws-demo</artifactId>
  <version>1.0-SNAPSHOT</version>
  
  <properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
    <aws.lambda.version>1.2.1</aws.lambda.version>
    <aws.sdk.version>2.17.112</aws.sdk.version>
  </properties>
  
  <dependencies>
    <!-- AWS Lambda -->
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-lambda-java-core</artifactId>
      <version>${aws.lambda.version}</version>
    </dependency>
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-lambda-java-events</artifactId>
      <version>3.11.0</version>
    </dependency>
    
    <!-- AWS SDK -->
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>sfn</artifactId>
      <version>${aws.sdk.version}</version>
    </dependency>
  </dependencies>
  
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.4</version>
        <configuration>
          <createDependencyReducedPom>false</createDependencyReducedPom>
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### 3. Terraform部署代码 (`terraform/main.tf`)

```hcl
provider "aws" {
  region = var.region
}

# Lambda函数
resource "aws_lambda_function" "start_execution" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "StartExecutionHandler"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.StartExecutionHandler"
  runtime       = "java11"
  memory_size   = 512
  timeout       = 30
  environment {
    variables = {
      STATE_MACHINE_ARN = aws_sfn_state_machine.chain.arn
      AWS_REGION        = var.region
    }
  }
}

resource "aws_lambda_function" "task1" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "StepFunctionTask1"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.StepFunctionTask1Handler"
  runtime       = "java11"
}

resource "aws_lambda_function" "task2" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "StepFunctionTask2"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.StepFunctionTask2Handler"
  runtime       = "java11"
}

resource "aws_lambda_function" "event_bridge_handler" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "EventBridgeHandler"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.EventBridgeHandler"
  runtime       = "java11"
}

# Step Functions状态机
resource "aws_sfn_state_machine" "chain" {
  name     = "ChainStepFunctions"
  role_arn = aws_iam_role.step_functions.arn
  definition = templatefile("${path.module}/../src/main/resources/statemachine.json", {
    task1_lambda_arn = aws_lambda_function.task1.arn
    task2_lambda_arn = aws_lambda_function.task2.arn
  })
}

# API Gateway
resource "aws_apigatewayv2_api" "main" {
  name          = "demo-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id           = aws_apigatewayv2_api.main.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.start_execution.invoke_arn
}

resource "aws_apigatewayv2_route" "post" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "POST /start"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_apigatewayv2_stage" "default" {
  api_id = aws_apigatewayv2_api.main.id
  name   = "$default"
}

# EventBridge规则
resource "aws_cloudwatch_event_rule" "step_function_success" {
  name        = "step-function-success"
  description = "Trigger when Step Functions completes"

  event_pattern = <<EOF
{
  "source": ["aws.states"],
  "detail-type": ["Step Functions Execution Status Change"],
  "detail": {
    "status": ["SUCCEEDED"],
    "stateMachineArn": ["${aws_sfn_state_machine.chain.arn}"]
  }
}
EOF
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.step_function_success.name
  target_id = "TriggerLambda"
  arn       = aws_lambda_function.event_bridge_handler.arn
}

# IAM角色
resource "aws_iam_role" "lambda_exec" {
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role" "step_functions" {
  name = "step_functions_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "states.amazonaws.com"
      }
    }]
  })
}

# 附加策略
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "step_functions" {
  name = "step_functions_policy"
  role = aws_iam_role.step_functions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action   = "lambda:InvokeFunction"
        Effect   = "Allow"
        Resource = "*"
      },
      {
        Action   = "states:StartExecution"
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}
```

### 4. 部署和使用说明

#### 部署步骤：
1. 打包Java项目：
```bash
mvn clean package
```

2. 初始化Terraform：
```bash
cd terraform
terraform init
```

3. 部署基础设施：
```bash
terraform apply -var="region=us-east-1"
```

#### 测试流程：
1. 调用API Gateway触发流程：
```bash
curl -X POST https://<api-gateway-url>/start -d '{"key":"value"}'
```

2. 流程执行：
   - API Gateway → 启动Lambda
   - Lambda → 启动Step Functions
   - Step Functions任务1 → Step Functions任务2
   - 成功完成后触发EventBridge事件
   - EventBridge → 触发最终Lambda

### 架构说明
1. **API Gateway**：接收HTTP请求并触发启动Lambda
2. **Lambda**：
   - 启动Lambda：启动Step Functions工作流
   - 任务Lambda：执行具体业务逻辑
   - 事件Lambda：处理EventBridge事件
3. **Step Functions**：链式执行多个Lambda任务
4. **EventBridge**：监听Step Functions完成事件并触发后续处理
5. **Terraform**：基础设施即代码部署所有资源

此实现展示了完整的Serverless编排流程，通过Terraform实现了一键部署，适合作为复杂业务场景的基础模板。

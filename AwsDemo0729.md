下面是一个完整的Java Maven项目，演示了使用AWS API Gateway、Lambda、Step Functions链式调用和EventBridge的服务集成，并通过Terraform部署，包含单元测试和详细注释。

### 项目结构
```bash
aws-demo/
├── src/
│   ├── main/
│   │   ├── java/com/example/
│   │   │   ├── ApiHandler.java         # API Gateway入口
│   │   │   ├── StepFunctionStarter.java # 启动Step Functions
│   │   │   ├── TaskHandler.java        # Step Functions任务处理
│   │   │   └── EventBridgeHandler.java # EventBridge处理器
│   │   └── resources/
│   └── test/
│       └── java/com/example/
│           └── HandlerTest.java        # 单元测试
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── pom.xml
```

---

### 1. Maven 依赖 (pom.xml)
```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>aws-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <dependencies>
        <!-- AWS Lambda -->
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-lambda-java-core</artifactId>
            <version>1.2.2</version>
        </dependency>
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-lambda-java-events</artifactId>
            <version>3.11.2</version>
        </dependency>
        
        <!-- Step Functions SDK -->
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-java-sdk-sfn</artifactId>
            <version>1.12.387</version>
        </dependency>
        
        <!-- JSON Processing -->
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.10.1</version>
        </dependency>
        
        <!-- Unit Testing -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>5.3.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.4.1</version>
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

---

### 2. Java 代码 (带详细注释)

#### 2.1 API Gateway 入口 (ApiHandler.java)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import com.google.gson.Gson;

/**
 * 处理API Gateway请求并触发Step Functions工作流
 */
public class ApiHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    private static final Gson GSON = new Gson();
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        // 1. 解析请求体
        String requestBody = input.getBody();
        InputModel data = GSON.fromJson(requestBody, InputModel.class);
        
        // 2. 触发Step Functions工作流
        StepFunctionStarter starter = new StepFunctionStarter();
        String executionArn = starter.startExecution(data);
        
        // 3. 返回响应
        return new APIGatewayProxyResponseEvent()
            .withStatusCode(200)
            .withBody("{\"executionArn\": \"" + executionArn + "\"}");
    }
    
    // 输入数据模型
    public static class InputModel {
        private String message;
        // 其他字段...
    }
}
```

#### 2.2 Step Functions 启动器 (StepFunctionStarter.java)
```java
package com.example;

import com.amazonaws.services.stepfunctions.AWSStepFunctions;
import com.amazonaws.services.stepfunctions.AWSStepFunctionsClientBuilder;
import com.amazonaws.services.stepfunctions.model.StartExecutionRequest;
import com.amazonaws.services.stepfunctions.model.StartExecutionResult;

/**
 * 启动Step Functions状态机执行
 */
public class StepFunctionStarter {

    // 状态机ARN通过环境变量注入
    private static final String STATE_MACHINE_ARN = System.getenv("STATE_MACHINE_ARN");
    
    public String startExecution(ApiHandler.InputModel input) {
        AWSStepFunctions client = AWSStepFunctionsClientBuilder.defaultClient();
        
        // 构建输入JSON
        String inputJson = "{\"message\": \"" + input.message + "\"}";
        
        // 启动状态机执行
        StartExecutionRequest request = new StartExecutionRequest()
            .withStateMachineArn(STATE_MACHINE_ARN)
            .withInput(inputJson);
        
        StartExecutionResult result = client.startExecution(request);
        return result.getExecutionArn();
    }
}
```

#### 2.3 Step Functions 任务处理器 (TaskHandler.java)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.google.gson.Gson;
import java.util.Map;

/**
 * Step Functions状态机的任务处理器
 */
public class TaskHandler implements RequestHandler<Map<String, Object>, String> {

    private static final Gson GSON = new Gson();
    
    @Override
    public String handleRequest(Map<String, Object> input, Context context) {
        // 1. 解析输入
        String message = (String) input.get("message");
        
        // 2. 处理业务逻辑
        String processedMessage = "Processed: " + message.toUpperCase();
        
        // 3. 返回结果（将作为下一步的输入）
        return processedMessage;
    }
}
```

#### 2.4 EventBridge 处理器 (EventBridgeHandler.java)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.ScheduledEvent;

/**
 * 处理EventBridge定时触发的事件
 */
public class EventBridgeHandler implements RequestHandler<ScheduledEvent, String> {

    @Override
    public String handleRequest(ScheduledEvent event, Context context) {
        // 处理定时任务逻辑
        return "EventBridge event processed at " + event.getTime();
    }
}
```

---

### 3. 单元测试 (HandlerTest.java)
```java
package com.example;

import static org.junit.Assert.*;
import static org.mockito.Mockito.*;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.stepfunctions.model.StartExecutionResult;
import java.util.HashMap;
import java.util.Map;
import org.junit.Test;
import org.mockito.MockedStatic;

public class HandlerTest {

    private final Context context = mock(Context.class);

    @Test
    public void testApiHandler() {
        // 准备输入
        ApiHandler.InputModel inputModel = new ApiHandler.InputModel();
        inputModel.message = "test";
        
        // 模拟StepFunctionStarter
        try (MockedStatic<StepFunctionStarter> mocked = mockStatic(StepFunctionStarter.class)) {
            StepFunctionStarter starter = mock(StepFunctionStarter.class);
            when(starter.startExecution(inputModel)).thenReturn("arn:test");
            mocked.when(StepFunctionStarter::new).thenReturn(starter);
            
            // 执行测试
            ApiHandler handler = new ApiHandler();
            APIGatewayProxyResponseEvent response = handler.handleRequest(
                new APIGatewayProxyRequestEvent().withBody("{\"message\":\"test\"}"), 
                context
            );
            
            // 验证结果
            assertEquals(200, response.getStatusCode());
            assertTrue(response.getBody().contains("arn:test"));
        }
    }

    @Test
    public void testTaskHandler() {
        // 准备输入
        Map<String, Object> input = new HashMap<>();
        input.put("message", "hello");
        
        // 执行测试
        TaskHandler handler = new TaskHandler();
        String result = handler.handleRequest(input, context);
        
        // 验证结果
        assertEquals("Processed: HELLO", result);
    }
}
```

---

### 4. Terraform 部署代码 (terraform/main.tf)

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# 1. 创建Lambda函数
resource "aws_lambda_function" "api_handler" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "ApiHandler"
  role          = aws_iam_role.lambda_role.arn
  handler       = "com.example.ApiHandler::handleRequest"
  runtime       = "java11"
}

resource "aws_lambda_function" "task_handler" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "TaskHandler"
  role          = aws_iam_role.lambda_role.arn
  handler       = "com.example.TaskHandler::handleRequest"
  runtime       = "java11"
}

# 2. 创建Step Functions状态机
resource "aws_sfn_state_machine" "demo_state_machine" {
  name     = "DemoStateMachine"
  role_arn = aws_iam_role.stepfunctions_role.arn

  definition = <<EOF
{
  "Comment": "链式调用示例",
  "StartAt": "FirstTask",
  "States": {
    "FirstTask": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.task_handler.arn}",
      "Next": "SecondTask"
    },
    "SecondTask": {
      "Type": "Task",
      "Resource": "${aws_lambda_function.task_handler.arn}",
      "End": true
    }
  }
}
EOF
}

# 3. 创建API Gateway
resource "aws_apigatewayv2_api" "demo_api" {
  name          = "demo-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda_integration" {
  api_id           = aws_apigatewayv2_api.demo_api.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.api_handler.invoke_arn
}

resource "aws_apigatewayv2_route" "post_route" {
  api_id    = aws_apigatewayv2_api.demo_api.id
  route_key = "POST /start"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}

# 4. 创建EventBridge规则
resource "aws_cloudwatch_event_rule" "daily_trigger" {
  name                = "daily-lambda-trigger"
  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "trigger_stepfunctions" {
  rule      = aws_cloudwatch_event_rule.daily_trigger.name
  target_id = "TriggerStepFunctions"
  arn       = aws_sfn_state_machine.demo_state_machine.arn
}

# 5. IAM角色配置
resource "aws_iam_role" "lambda_role" {
  name = "lambda_execution_role"

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

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role" "stepfunctions_role" {
  name = "stepfunctions_execution_role"

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

resource "aws_iam_role_policy" "stepfunctions_policy" {
  name = "invoke_lambda_policy"
  role = aws_iam_role.stepfunctions_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "lambda:InvokeFunction"
      Effect   = "Allow"
      Resource = aws_lambda_function.task_handler.arn
    }]
  })
}
```

---

### 5. 部署与运行流程

#### 5.1 本地构建与测试
```bash
# 构建项目
mvn clean package

# 运行单元测试
mvn test
```

#### 5.2 Terraform 部署
```bash
cd terraform
terraform init
terraform apply -auto-approve
```

#### 5.3 测试API
```bash
curl -X POST https://<api-gateway-url>/start \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello AWS"}'
```

#### 5.4 验证工作流
1. 在AWS控制台查看Step Functions执行历史
2. 检查CloudWatch日志中的Lambda输出
3. 验证EventBridge每日触发状态机

---

### 架构说明
1. **API Gateway** 接收HTTP请求
2. **Lambda (ApiHandler)** 启动Step Functions工作流
3. **Step Functions** 链式调用Lambda任务处理器
4. **EventBridge** 定时触发状态机执行
5. **Terraform** 统一管理所有基础设施

该演示完整覆盖了AWS无服务器核心服务，通过Java实现业务逻辑，Terraform实现IaC，适合作为复杂工作流的入门模板。

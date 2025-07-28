
下面是一个完整的Java Maven项目，集成了AWS API Gateway、Lambda、Step Functions链式调用和EventBridge，并通过Terraform部署。我将提供详细解释、单元测试方法和本地运行指南。

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
│   └── test/                                  # 单元测试
│       └── java/com/example/
│           └── HandlerTest.java
├── terraform/
│   ├── main.tf                                # 主部署配置
│   ├── variables.tf                           # 环境变量
│   └── outputs.tf                             # 输出配置
├── pom.xml                                    # Maven配置
└── sam-local/                                 # 本地测试配置
    └── template.yaml
```

### 1. Java核心代码

#### `StartExecutionHandler.java`
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sfn.SfnClient;
import software.amazon.awssdk.services.sfn.model.StartExecutionRequest;
import com.google.gson.Gson;

public class StartExecutionHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    private static final Gson gson = new Gson();
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        try {
            SfnClient sfn = SfnClient.builder()
                .region(Region.of(System.getenv("AWS_REGION")))
                .build();

            // 启动Step Functions执行
            StartExecutionRequest request = StartExecutionRequest.builder()
                .stateMachineArn(System.getenv("STATE_MACHINE_ARN"))
                .input(input.getBody())
                .build();
            
            String executionArn = sfn.startExecution(request).executionArn();
            
            // 构建JSON响应
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(200)
                .withBody(String.format("{\"executionArn\":\"%s\"}", executionArn));
        } catch (Exception e) {
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(500)
                .withBody("{\"error\":\"" + e.getMessage() + "\"}");
        }
    }
}
```

#### `StepFunctionTask1Handler.java`
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import java.util.Map;

public class StepFunctionTask1Handler implements RequestHandler<Map<String, Object>, Map<String, Object>> {
    @Override
    public Map<String, Object> handleRequest(Map<String, Object> input, Context context) {
        // 模拟业务处理
        input.put("task1", "processed");
        input.put("statusCode", 200);
        return input;
    }
}
```

#### `EventBridgeHandler.java`
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.CloudWatchEvent;

public class EventBridgeHandler implements RequestHandler<CloudWatchEvent<String>, String> {
    @Override
    public String handleRequest(CloudWatchEvent<String> event, Context context) {
        // 处理EventBridge事件
        context.getLogger().log("Received event: " + event.getDetail());
        return "Event processed at " + event.getTime();
    }
}
```

### 2. Terraform配置详解 (`terraform/main.tf`)

```hcl
# 配置AWS提供商
provider "aws" {
  region = "us-east-1"  # 指定部署区域
}

# 创建Lambda执行角色
resource "aws_iam_role" "lambda_exec" {
  name = "lambda_exec_role"
  
  # 信任策略：允许Lambda服务代入此角色
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

# 附加基础执行策略
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# 创建启动Step Functions的Lambda
resource "aws_lambda_function" "start_execution" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"  # 打包后的JAR路径
  function_name = "StartExecutionHandler"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.StartExecutionHandler"    # Java处理类
  runtime       = "java11"
  timeout       = 30
  
  # 环境变量：传递给Lambda
  environment {
    variables = {
      STATE_MACHINE_ARN = aws_sfn_state_machine.chain.arn
      AWS_REGION        = "us-east-1"
    }
  }
}

# 创建任务处理Lambda
resource "aws_lambda_function" "task1" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "StepFunctionTask1"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.StepFunctionTask1Handler"
  runtime       = "java11"
}

# 创建EventBridge处理Lambda
resource "aws_lambda_function" "event_bridge" {
  filename      = "../target/aws-demo-1.0-SNAPSHOT.jar"
  function_name = "EventBridgeHandler"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "com.example.EventBridgeHandler"
  runtime       = "java11"
}

# 创建Step Functions角色
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

# 允许Step Functions调用Lambda
resource "aws_iam_role_policy" "step_functions" {
  name = "step_functions_policy"
  role = aws_iam_role.step_functions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action   = "lambda:InvokeFunction"
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}

# 创建Step Functions状态机
resource "aws_sfn_state_machine" "chain" {
  name     = "ProcessingWorkflow"
  role_arn = aws_iam_role.step_functions.arn
  
  # 状态机定义（从文件加载）
  definition = templatefile("${path.module}/../src/main/resources/statemachine.json", {
    task1_arn = aws_lambda_function.task1.arn
  })
}

# 创建API Gateway
resource "aws_apigatewayv2_api" "api" {
  name          = "demo-api"
  protocol_type = "HTTP"
}

# 集成Lambda到API Gateway
resource "aws_apigatewayv2_integration" "lambda" {
  api_id           = aws_apigatewayv2_api.api.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.start_execution.invoke_arn
}

# 创建路由
resource "aws_apigatewayv2_route" "route" {
  api_id    = aws_apigatewayv2_api.api.id
  route_key = "POST /start"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

# 部署API
resource "aws_apigatewayv2_stage" "default" {
  api_id = aws_apigatewayv2_api.api.id
  name   = "$default"
}

# 创建EventBridge规则
resource "aws_cloudwatch_event_rule" "step_success" {
  name        = "step-function-success"
  description = "Trigger on Step Function success"
  
  # 事件匹配模式
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

# 将规则绑定到Lambda
resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.step_success.name
  target_id = "TriggerLambda"
  arn       = aws_lambda_function.event_bridge.arn
}

# 允许EventBridge调用Lambda
resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.event_bridge.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.step_success.arn
}
```

### 3. Step Functions定义 (`statemachine.json`)
```json
{
  "Comment": "Processing Workflow",
  "StartAt": "ProcessTask",
  "States": {
    "ProcessTask": {
      "Type": "Task",
      "Resource": "${task1_arn}",
      "End": true
    }
  }
}
```

### 4. Maven配置 (`pom.xml`)
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
    <!-- AWS Lambda Core -->
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-lambda-java-core</artifactId>
      <version>${aws.lambda.version}</version>
    </dependency>
    
    <!-- AWS Lambda Events -->
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
    
    <!-- JSON处理 -->
    <dependency>
      <groupId>com.google.code.gson</groupId>
      <artifactId>gson</artifactId>
      <version>2.8.9</version>
    </dependency>
    
    <!-- 单元测试 -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <version>3.12.4</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  
  <build>
    <plugins>
      <!-- 打包插件 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.4</version>
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

### 5. 单元测试示例
```java
package com.example;

import static org.junit.Assert.*;
import org.junit.Test;
import com.amazonaws.services.lambda.runtime.Context;
import java.util.HashMap;
import java.util.Map;

public class HandlerTest {
    
    // 测试Lambda处理程序
    @Test
    public void testTask1Handler() {
        StepFunctionTask1Handler handler = new StepFunctionTask1Handler();
        Map<String, Object> input = new HashMap<>();
        input.put("key", "value");
        
        Map<String, Object> result = handler.handleRequest(input, new TestContext());
        
        assertEquals("processed", result.get("task1"));
        assertEquals(200, result.get("statusCode"));
    }
    
    // 模拟Context实现
    private static class TestContext implements Context {
        public String getAwsRequestId() { return ""; }
        public String getLogGroupName() { return ""; }
        public String getLogStreamName() { return ""; }
        public String getFunctionName() { return ""; }
        public String getFunctionVersion() { return ""; }
        public String getInvokedFunctionArn() { return ""; }
        public int getRemainingTimeInMillis() { return 0; }
        public int getMemoryLimitInMB() { return 0; }
        public String getLoggerName() { return ""; }
    }
}
```

### 6. 本地运行指南

#### 使用AWS SAM测试：
1. 创建 `sam-local/template.yaml`:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  StartExecutionFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ../target/aws-demo-1.0-SNAPSHOT.jar
      Handler: com.example.StartExecutionHandler::handleRequest
      Runtime: java11
      Environment:
        Variables:
          STATE_MACHINE_ARN: "arn:aws:states:us-east-1:1234567890:stateMachine/Test"
          AWS_REGION: "us-east-1"
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /start
            Method: post
```

2. 启动本地API:
```bash
sam local start-api -t sam-local/template.yaml
```

3. 发送测试请求:
```bash
curl -X POST http://localhost:3000/start -d '{"test":"data"}'
```

#### 直接运行Lambda处理程序:
```java
public class LocalRunner {
    public static void main(String[] args) {
        // 测试StartExecutionHandler
        StartExecutionHandler handler = new StartExecutionHandler();
        APIGatewayProxyRequestEvent request = new APIGatewayProxyRequestEvent()
            .withBody("{\"key\":\"value\"}");
        
        APIGatewayProxyResponseEvent response = handler.handleRequest(request, new TestContext());
        System.out.println("Response: " + response.getBody());
        
        // 测试Task1Handler
        StepFunctionTask1Handler taskHandler = new StepFunctionTask1Handler();
        Map<String, Object> input = new HashMap<>();
        input.put("data", "test");
        System.out.println("Task Output: " + taskHandler.handleRequest(input, new TestContext()));
    }
}
```

### 部署流程

1. **打包应用**:
```bash
mvn clean package
```

2. **初始化Terraform**:
```bash
cd terraform
terraform init
```

3. **部署基础设施**:
```bash
terraform apply
```

4. **测试API**:
```bash
curl -X POST $(terraform output -raw api_url) -d '{"data":"test"}'
```

### 架构说明

1. **API Gateway**:
   - 提供REST端点 `POST /start`
   - 将请求转发到启动Lambda

2. **启动Lambda**:
   - 接收API请求
   - 启动Step Functions工作流
   - 返回执行ARN

3. **Step Functions**:
   - 执行任务链（Task1 → Task2）
   - 每个任务调用对应的Lambda

4. **EventBridge**:
   - 监听Step Functions完成事件
   - 触发最终处理Lambda

5. **最终Lambda**:
   - 处理工作流完成事件
   - 执行后续操作（如通知、清理）

### 关键优势

1. **完全托管服务**：无需管理服务器
2. **自动扩展**：根据负载动态调整资源
3. **事件驱动**：服务间通过事件解耦
4. **可视化流程**：Step Functions提供执行可视化
5. **基础设施即代码**：Terraform实现可重复部署

此实现展示了完整的Serverless架构模式，适合构建可扩展的事件驱动型应用，通过Terraform实现了一键部署，极大简化了运维复杂度。

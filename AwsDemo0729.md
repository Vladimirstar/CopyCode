以下是一个完整的AWS Java SDK (JDK 21) Demo，包含Lambda、Step Functions、EventBridge和API Gateway，使用Maven构建，并通过Terraform部署：

### 项目结构
```
aws-java-demo/
├── src/
│   ├── main/
│   │   ├── java/com/example/
│   │   │   ├── App.java                # Lambda处理程序
│   │   │   ├── StepFunctionClient.java # Step Functions工具
│   │   │   └── EventBridgeClient.java  # EventBridge工具
│   │   └── resources/
│   └── test/
│       └── java/com/example/
│           └── AppTest.java            # 单元测试
├── terraform/
│   ├── main.tf                         # Terraform配置
│   ├── variables.tf                    # Terraform变量
│   └── outputs.tf                      # Terraform输出
├── pom.xml                             # Maven配置
└── statemachine.json                   # Step Functions定义
```

### 1. Maven配置 (pom.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>aws-java-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <aws.sdk.version>2.17.112</aws.sdk.version>
        <junit.version>5.9.2</junit.version>
    </properties>

    <dependencies>
        <!-- AWS SDK -->
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>lambda</artifactId>
            <version>${aws.sdk.version}</version>
        </dependency>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>stepfunctions</artifactId>
            <version>${aws.sdk.version}</version>
        </dependency>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>eventbridge</artifactId>
            <version>${aws.sdk.version}</version>
        </dependency>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>apigateway</artifactId>
            <version>${aws.sdk.version}</version>
        </dependency>
        
        <!-- Lambda Runtime -->
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-lambda-java-core</artifactId>
            <version>1.2.2</version>
        </dependency>
        
        <!-- Unit Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>4.8.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.3.0</version>
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

### 2. Lambda处理程序 (App.java)
```java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import java.util.Map;

public class App implements RequestHandler<Map<String, Object>, String> {

    @Override
    public String handleRequest(Map<String, Object> input, Context context) {
        // 1. 从API Gateway接收请求
        System.out.println("Received input: " + input);
        
        // 2. 触发Step Functions
        String executionArn = StepFunctionClient.startExecution(
            "arn:aws:states:us-east-1:123456789012:stateMachine:MyStateMachine",
            "{\"key\":\"value\"}"
        );
        
        // 3. 发送EventBridge事件
        EventBridgeClient.putEvent(
            "demo-bus",
            "DemoEventType",
            "{\"status\":\"STARTED\",\"executionArn\":\"" + executionArn + "\"}"
        );
        
        return "{\"status\":\"success\",\"executionArn\":\"" + executionArn + "\"}";
    }
}
```

### 3. Step Functions工具类 (StepFunctionClient.java)
```java
package com.example;

import software.amazon.awssdk.services.stepfunctions.SfnClient;
import software.amazon.awssdk.services.stepfunctions.model.StartExecutionRequest;
import software.amazon.awssdk.services.stepfunctions.model.StartExecutionResponse;

public class StepFunctionClient {
    public static String startExecution(String stateMachineArn, String input) {
        try (SfnClient sfnClient = SfnClient.create()) {
            StartExecutionRequest request = StartExecutionRequest.builder()
                .stateMachineArn(stateMachineArn)
                .input(input)
                .build();
            
            StartExecutionResponse response = sfnClient.startExecution(request);
            return response.executionArn();
        }
    }
}
```

### 4. EventBridge工具类 (EventBridgeClient.java)
```java
package com.example;

import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequest;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequestEntry;

public class EventBridgeClient {
    public static void putEvent(String eventBusName, String detailType, String detail) {
        try (EventBridgeClient ebClient = EventBridgeClient.create()) {
            PutEventsRequestEntry entry = PutEventsRequestEntry.builder()
                .eventBusName(eventBusName)
                .source("com.example.demo")
                .detailType(detailType)
                .detail(detail)
                .build();
            
            ebClient.putEvents(PutEventsRequest.builder()
                .entries(entry)
                .build());
        }
    }
}
```

### 5. 单元测试 (AppTest.java)
```java
package com.example;

import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;
import org.mockito.Mockito;
import java.util.Collections;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;

class AppTest {
    @Test
    void testLambdaHandler() {
        // Mock静态方法
        try (MockedStatic<StepFunctionClient> mockedSfn = Mockito.mockStatic(StepFunctionClient.class);
             MockedStatic<EventBridgeClient> mockedEb = Mockito.mockStatic(EventBridgeClient.class)) {
            
            // 配置模拟行为
            mockedSfn.when(() -> StepFunctionClient.startExecution(anyString(), anyString()))
                     .thenReturn("arn:aws:states:::execution-arn");
            
            // 执行测试
            App app = new App();
            String result = app.handleRequest(Collections.singletonMap("key", "value"), null);
            
            // 验证结果
            assertTrue(result.contains("success"));
            assertTrue(result.contains("execution-arn"));
            
            // 验证方法调用
            mockedSfn.verify(() -> StepFunctionClient.startExecution(anyString(), anyString()));
            mockedEb.verify(() -> EventBridgeClient.putEvent(anyString(), anyString(), anyString()));
        }
    }
}
```

### 6. Step Functions定义 (statemachine.json)
```json
{
  "Comment": "Demo State Machine",
  "StartAt": "HelloWorld",
  "States": {
    "HelloWorld": {
      "Type": "Pass",
      "Result": "Hello from Step Functions!",
      "End": true
    }
  }
}
```

### 7. Terraform配置 (terraform/main.tf)
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

# Lambda函数
resource "aws_lambda_function" "demo_lambda" {
  filename      = "../target/aws-java-demo-1.0-SNAPSHOT.jar"
  function_name = "java-demo-handler"
  role          = aws_iam_role.lambda_role.arn
  handler       = "com.example.App::handleRequest"
  runtime       = "java21"
  memory_size   = 512
  timeout       = 30
}

# Step Functions
resource "aws_sfn_state_machine" "demo_sfn" {
  name     = "MyStateMachine"
  role_arn = aws_iam_role.sfn_role.arn
  definition = file("../statemachine.json")
}

# EventBridge总线
resource "aws_cloudwatch_event_bus" "demo_bus" {
  name = "demo-bus"
}

# API Gateway
resource "aws_apigatewayv2_api" "demo_api" {
  name          = "demo-http-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda_integration" {
  api_id           = aws_apigatewayv2_api.demo_api.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.demo_lambda.invoke_arn
}

resource "aws_apigatewayv2_route" "demo_route" {
  api_id    = aws_apigatewayv2_api.demo_api.id
  route_key = "GET /trigger"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}

# IAM角色
resource "aws_iam_role" "lambda_role" {
  name = "lambda-execution-role"
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
  managed_policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole",
    "arn:aws:iam::aws:policy/AWSStepFunctionsFullAccess",
    "arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess"
  ]
}

resource "aws_iam_role" "sfn_role" {
  name = "stepfunctions-execution-role"
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
  managed_policy_arns = [
    "arn:aws:iam::aws:policy/AWSLambda_FullAccess"
  ]
}
```

### 8. 部署和运行步骤

**1. 构建项目**
```bash
mvn clean package
```

**2. 初始化Terraform**
```bash
cd terraform
terraform init
```

**3. 部署到AWS**
```bash
terraform apply -auto-approve
```

**4. 本地测试Lambda**
```bash
# 使用SAM CLI本地测试
sam local invoke -e event.json

# event.json示例:
{
  "queryStringParameters": {
    "name": "test"
  }
}
```

**5. 调用API Gateway端点**
```bash
# 获取API端点URL
terraform output api_url

# 发送请求
curl "<api_url>/trigger"
```

**6. 验证流程**
1. API Gateway触发Lambda
2. Lambda启动Step Functions执行
3. Lambda发送EventBridge事件
4. 在AWS控制台查看:
   - Step Functions执行历史
   - EventBridge事件总线
   - CloudWatch日志

### 架构说明
1. **API Gateway**：接收HTTP请求并触发Lambda
2. **Lambda**：Java处理程序协调整个流程
3. **Step Functions**：执行定义的工作流
4. **EventBridge**：事件总线用于服务间通信

### 注意事项
1. 确保AWS凭证已配置（`~/.aws/credentials`）
2. JDK 21需预先安装
3. Terraform v1.0+ 和 AWS CLI v2+ 需预先安装
4. 测试时需根据实际AWS账户修改ARN
5. 使用`terraform destroy`清理资源

这个Demo展示了AWS服务间的集成，使用Java 21现代特性，并通过Terraform实现基础设施即代码(IaC)。

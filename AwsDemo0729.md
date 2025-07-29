
以下是一个使用JDK 21的AWS综合示例，涵盖API Gateway、Lambda、Step Functions和EventBridge，包含Maven项目配置、单元测试和Terraform部署脚本。所有代码均附带详细注释。

---

 项目结构
bash
aws-jdk21-demo/
├── pom.xml                   Maven配置
├── src
│   ├── main
│   │   ├── java/com/example/
│   │   │   ├── ApiHandler.java          API Gateway触发的Lambda
│   │   │   ├── TaskHandler.java         Step Functions任务处理器
│   │   │   └── EventBridgeHandler.java  EventBridge事件处理器
│   │   └── resources
│   │       └── statemachine.json        Step Functions状态机定义
│   └── test/java/com/example
│       └── HandlerTest.java             单元测试
└── infrastructure
    ├── main.tf              Terraform部署配置
    ├── variables.tf         Terraform变量
    └── outputs.tf           Terraform输出

---

 核心代码实现

 1. Maven配置 (`pom.xml`)
xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>aws-jdk21-demo</artifactId>
  <version>1.0-SNAPSHOT</version>
  
  <properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <aws.sdk.version>2.31.17</aws.sdk.version>
  </properties>
  
  <dependencies>
    <!-- AWS Lambda Core -->
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-lambda-java-core</artifactId>
      <version>1.2.3</version>
    </dependency>
    
    <!-- AWS SDK (Step Functions, EventBridge) -->
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>sfn</artifactId>
      <version>${aws.sdk.version}</version>
    </dependency>
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>eventbridge</artifactId>
      <version>${aws.sdk.version}</version>
    </dependency>
    
    <!-- JUnit 5 -->
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.10.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>

 2. Lambda处理器 (`ApiHandler.java`)
java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import software.amazon.awssdk.services.sfn.SfnClient;
import software.amazon.awssdk.services.sfn.model.StartExecutionRequest;

/
 * 接收API Gateway请求，触发Step Functions状态机
 * 输入示例: {"name": "World"}
 */
public class ApiHandler implements RequestHandler<Map<String, String>, String> {
    @Override
    public String handleRequest(Map<String, String> event, Context context) {
        // 1. 解析API Gateway输入
        String name = event.getOrDefault("name", "Guest");
        
        // 2. 启动Step Functions执行
        try (SfnClient sfnClient = SfnClient.create()) {
            StartExecutionRequest request = StartExecutionRequest.builder()
                .stateMachineArn(System.getenv("STATE_MACHINE_ARN")) // 从环境变量获取ARN
                .input("{\"name\": \"" + name + "\"}") // 传递参数
                .build();
            
            sfnClient.startExecution(request);
            return "Workflow started for: " + name;
        }
    }
}

 3. Step Functions任务 (`TaskHandler.java`)
java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

/
 * Step Functions状态机调用的任务
 * 输入示例: {"name": "World"}
 */
public class TaskHandler implements RequestHandler<Map<String, String>, String> {
    @Override
    public String handleRequest(Map<String, String> event, Context context) {
        String name = event.get("name");
        return "Hello, " + name + " from Step Function!";
    }
}

 4. EventBridge处理器 (`EventBridgeHandler.java`)
java
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequest;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequestEntry;

/
 * 定时触发Step Functions（通过EventBridge规则）
 */
public class EventBridgeHandler implements RequestHandler<Object, String> {
    @Override
    public String handleRequest(Object event, Context context) {
        try (EventBridgeClient eventBridgeClient = EventBridgeClient.create()) {
            PutEventsRequestEntry entry = PutEventsRequestEntry.builder()
                .source("aws.demo")
                .detailType("ScheduledEvent")
                .detail("{\"name\": \"ScheduledUser\"}")
                .build();
            
            eventBridgeClient.putEvents(PutEventsRequest.builder()
                .entries(entry)
                .build());
            
            return "Scheduled event triggered";
        }
    }
}

 5. Step Functions状态机定义 (`statemachine.json`)
json
{
  "Comment": "演示工作流",
  "StartAt": "HelloStep",
  "States": {
    "HelloStep": {
      "Type": "Task",
      "Resource": "${LAMBDA_TASK_ARN}", // Terraform替换为实际ARN
      "End": true
    }
  }
}

---

 单元测试 (`HandlerTest.java`)
java
package com.example;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class HandlerTest {
    @Test
    void testApiHandler() {
        ApiHandler handler = new ApiHandler();
        Map<String, String> input = Map.of("name", "TestUser");
        String result = handler.handleRequest(input, null);
        assertTrue(result.contains("Workflow started"));
    }

    @Test
    void testTaskHandler() {
        TaskHandler handler = new TaskHandler();
        Map<String, String> input = Map.of("name", "TestUser");
        String result = handler.handleRequest(input, null);
        assertEquals("Hello, TestUser from Step Function!", result);
    }
}

---

 Terraform部署脚本 (`infrastructure/`)

 1. 主配置 (`main.tf`)
hcl
provider "aws" {
  region = "us-west-2"
}

 创建Lambda函数
resource "aws_lambda_function" "api_handler" {
  filename      = "target/aws-jdk21-demo.jar"
  function_name = "ApiHandler"
  handler       = "com.example.ApiHandler::handleRequest"
  runtime       = "java21"
  role          = aws_iam_role.lambda_role.arn
  environment {
    variables = {
      STATE_MACHINE_ARN = aws_sfn_state_machine.demo.arn
    }
  }
}

 Step Functions状态机
resource "aws_sfn_state_machine" "demo" {
  name     = "HelloWorkflow"
  role_arn = aws_iam_role.step_role.arn
  definition = templatefile("${path.module}/../src/main/resources/statemachine.json", {
    LAMBDA_TASK_ARN = aws_lambda_function.task_handler.arn
  })
}

 EventBridge规则
resource "aws_cloudwatch_event_rule" "daily" {
  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "trigger_lambda" {
  rule = aws_cloudwatch_event_rule.daily.name
  arn  = aws_lambda_function.event_bridge_handler.arn
}

 IAM角色配置（省略部分策略）
resource "aws_iam_role" "lambda_role" {
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = {
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }
  })
}

 输出API Gateway URL
output "api_url" {
  value = aws_api_gateway_deployment.demo.invoke_url
}

 2. 部署步骤
bash
 1. 构建项目
mvn clean package

 2. 初始化Terraform
cd infrastructure
terraform init

 3. 部署到AWS
terraform apply -auto-approve

 4. 获取测试URL
echo "API Gateway URL: $(terraform output api_url)"

---

 本地运行验证
1. 模拟Lambda调用：
   java
   public class LocalTest {
       public static void main(String args) {
           ApiHandler handler = new ApiHandler();
           System.out.println(handler.handleRequest(Map.of("name", "LocalUser"), null));
       }
   }

2. 测试EventBridge触发：
   bash
   aws events put-events --entries '
     {"Source":"aws.demo", "DetailType":"ScheduledEvent", "Detail":"{\"name\":\"TestUser\"}"}
   ' --region us-west-2

---

 架构解析
1. API Gateway：接收HTTP请求，触发`ApiHandler` Lambda。
2. Lambda：
   - `ApiHandler`：启动Step Functions工作流。
   - `TaskHandler`：执行具体业务逻辑。
   - `EventBridgeHandler`：响应定时事件。
3. Step Functions：协调任务执行顺序。
4. EventBridge：定时触发Lambda（每日1次）。
5. Terraform：实现IaC（基础设施即代码）部署。

> 提示**：完整代码参考和。

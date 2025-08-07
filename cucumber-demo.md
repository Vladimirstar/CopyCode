下面是一个简单的 Cucumber BDD（行为驱动开发）示例，帮助你快速入门。这个示例模拟用户登录功能，包含 Feature 文件、Step Definitions 和 Runner 类。

### 1. 环境准备
确保项目中包含以下依赖（Maven 示例）：
```xml
<dependencies>
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-java</artifactId>
        <version>7.15.0</version>
    </dependency>
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit</artifactId>
        <version>7.15.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

### 2. Feature 文件 (Gherkin语法)
创建文件：`src/test/resources/features/login.feature`
```gherkin
Feature: 用户登录功能
  作为用户
  我需要登录系统
  以便访问个人账户

  Scenario: 使用正确密码登录成功
    Given 用户打开登录页面
    When 输入用户名 "testuser"
    And 输入密码 "Pass123"
    And 点击登录按钮
    Then 应显示欢迎消息 "欢迎回来, testuser!"

  Scenario: 使用错误密码登录失败
    Given 用户打开登录页面
    When 输入用户名 "testuser"
    And 输入密码 "wrongPass"
    And 点击登录按钮
    Then 应显示错误消息 "密码错误!"
```

---

### 3. Step Definitions (Java实现)
创建文件：`src/test/java/steps/LoginSteps.java`
```java
package steps;

import io.cucumber.java.zh_cn.*;
import static org.junit.Assert.*;

public class LoginSteps {
    
    private String username;
    private String password;
    private String actualMessage;

    @Given("用户打开登录页面")
    public void openLoginPage() {
        // 模拟打开页面操作
        System.out.println("打开登录页面");
    }

    @When("输入用户名 {string}")
    public void enterUsername(String username) {
        this.username = username;
    }

    @When("输入密码 {string}")
    public void enterPassword(String password) {
        this.password = password;
    }

    @When("点击登录按钮")
    public void clickLoginButton() {
        // 模拟登录逻辑
        if ("Pass123".equals(password)) {
            actualMessage = "欢迎回来, " + username + "!";
        } else {
            actualMessage = "密码错误!";
        }
    }

    @Then("应显示欢迎消息 {string}")
    public void verifyWelcomeMessage(String expectedMessage) {
        assertEquals(expectedMessage, actualMessage);
    }

    @Then("应显示错误消息 {string}")
    public void verifyErrorMessage(String expectedMessage) {
        assertEquals(expectedMessage, actualMessage);
    }
}
```

---

### 4. Test Runner (JUnit 运行器)
创建文件：`src/test/java/runners/TestRunner.java`
```java
package runners;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
    features = "src/test/resources/features", // Feature文件路径
    glue = "steps",                          // Step Definitions包名
    plugin = {"pretty", "html:target/cucumber-report.html"} // 生成HTML报告
)
public class TestRunner {
}
```

---

### 5. 运行测试
使用以下方式运行：
1. **IDE**：右键运行 `TestRunner.java`
2. **Maven**：执行命令
   ```bash
   mvn test
   ```

---

### 6. 查看结果
执行后会生成报告：
```
打开登录页面
输入用户名 testuser
输入密码 Pass123
点击登录按钮
验证消息：欢迎回来, testuser!

打开登录页面
输入用户名 testuser
输入密码 wrongPass
点击登录按钮
验证消息：密码错误!
```
报告路径：`target/cucumber-report.html`

---

### 关键概念解释
1. **Feature 文件**：用自然语言描述功能（.feature 后缀）
2. **Scenario**：具体测试场景（一个Feature包含多个Scenario）
3. **Step Definitions**：将Gherkin步骤映射为Java代码
4. **Runner**：配置并启动测试的JUnit运行器
5. **Gherkin 关键字**：
   - `Given`：前置条件
   - `When`：触发事件
   - `Then`：预期结果
   - `And`：连接多个操作

---

### 扩展练习建议
1. 添加新场景：测试用户名不存在的情况
2. 使用数据表示例（Scenario Outline）：
```gherkin
Scenario Outline: 不同用户登录
  Given 用户打开登录页面
  When 输入用户名 "<username>"
  And 输入密码 "<password>"
  And 点击登录按钮
  Then 应显示消息 "<message>"

  Examples:
    | username  | password  | message               |
    | testuser1 | Pass123   | 欢迎回来, testuser1! |
    | testuser2 | wrongPass | 密码错误!             |
```

通过这个简单示例，你可以快速理解 Cucumber 的核心工作流程。实际项目中可以结合 Selenium 做 Web 自动化测试！

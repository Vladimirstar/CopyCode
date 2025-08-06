要清空 JDK 21 的证书（即清除其默认信任库 `cacerts` 中的所有证书），可通过以下方法操作。**注意**：清空证书会导致 JDK 无法验证任何 HTTPS 站点的合法性，仅建议在测试或重建信任链时使用，操作前务必备份原始文件‼️

---

### ⚙️ 方法一：直接替换 `cacerts` 文件（推荐）
此方法通过删除或重命名原始信任库文件，触发 JDK 自动生成一个**空的新信任库**：
1. **定位 JDK 21 的 `cacerts` 文件路径**  
   默认位置：  
   ```bash
   ${JAVA_HOME}/lib/security/cacerts  # JDK 9 及以上路径
   ```
   > 注：JDK 8 及更早版本路径为 `${JAVA_HOME}/jre/lib/security/cacerts`，但 JDK 21 适用前者。

2. **备份原始文件（防止误操作）**  
   ```bash
   cp cacerts cacerts.backup  # Linux/macOS
   copy cacerts cacerts.backup # Windows
   ```

3. **删除或重命名 `cacerts`**  
   ```bash
   rm cacerts  # Linux/macOS
   del cacerts # Windows
   ```
   - **效果**：重启 Java 应用时，JDK 会自动生成一个**空的 `cacerts` 文件**，原有证书全部清空。

---

### ⚙️ 方法二：使用 `keytool` 删除所有证书
若需保留文件结构但逐条删除证书，可用以下命令：
1. **列出所有证书别名**  
   ```bash
   keytool -list -keystore ${JAVA_HOME}/lib/security/cacerts -storepass changeit
   ```
2. **批量删除证书**  
   通过脚本循环删除所有别名（以 Linux 为例）：
   ```bash
   keytool -list -keystore cacerts -storepass changeit | grep "别名类型," | awk '{print $1}' | xargs -I {} keytool -delete -alias {} -keystore cacerts -storepass changeit
   ```
   > **注意**：  
   > - Windows 需改用 PowerShell 或 Batch 脚本实现类似循环；  
   > - 若遇到系统内置证书（如 `verisign`）无法删除，属正常保护机制。

---

### ⚙️ 方法三：重新生成新信任库文件
彻底重建一个空的信任库：
```bash
keytool -genkeypair -alias empty -keyalg RSA -keystore cacerts_new -storepass changeit -dname "CN=Temp, OU=Temp, O=Temp, L=Temp, ST=Temp, C=Temp"
keytool -delete -alias empty -keystore cacerts_new -storepass changeit
```
随后将生成的 `cacerts_new` 替换原 `cacerts` 文件。

---

### ⚠️ 操作后注意事项
1. **恢复默认证书**：若需还原，用备份的 `cacerts.backup` 覆盖即可；  
2. **影响范围**：清空后所有依赖 JDK 信任库的 HTTPS 连接（如 Maven 拉包、API 调用）均会因证书验证失败报错；  
3. **临时解决方案**：在代码中临时跳过证书验证（仅限开发环境）：  
   ```java
   System.setProperty("javax.net.ssl.trustStore", "/path/to/empty/cacerts");
   System.setProperty("javax.net.ssl.trustStorePassword", "changeit");
   
   ```

---

### 📌 总结建议
- **首选方法一**：直接删除 `cacerts` 让 JDK 自动生成空文件，操作简单且彻底；  
- **生产环境慎用**：清空证书将导致所有安全连接失效，仅限本地开发或测试环境使用；  
- **恢复默认证书**：从其他正常 JDK 21 实例中复制 `cacerts` 文件覆盖即可。  

> 💡 更多 `keytool` 命令参考：[JDK keytool 文档](https://docs.oracle.com/javase/21/docs/specs/man/keytool.html)。

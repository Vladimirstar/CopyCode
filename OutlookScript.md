我来帮您写一个Python脚本，可以定时收集Outlook邮件到Excel文件。您可以选择以下几种方案：

## 方案一：Python + Outlook COM接口（推荐）

```python
import win32com.client
import pandas as pd
import schedule
import time
from datetime import datetime, timedelta
import os

def export_emails_to_excel():
    try:
        # 连接到Outlook
        outlook = win32com.client.Dispatch("Outlook.Application")
        namespace = outlook.GetNamespace("MAPI")
        inbox = namespace.GetDefaultFolder(6)  # 6代表收件箱
        
        # 获取今天的邮件（可以调整时间范围）
        yesterday = datetime.now() - timedelta(days=1)
        restrict_items = inbox.Items
        restrict_items = restrict_items.Restrict(f"[ReceivedTime] >= '{yesterday.strftime('%m/%d/%Y')}'")
        
        emails_data = []
        
        # 遍历邮件并提取信息
        for item in restrict_items:
            try:
                email_data = {
                    '发件人': item.SenderEmailAddress if item.SenderEmailAddress else str(item.Sender),
                    '发件人姓名': item.SenderName,
                    '主题': item.Subject,
                    '接收时间': item.ReceivedTime.strftime('%Y-%m-%d %H:%M:%S'),
                    '重要性': item.Importance,
                    '是否有附件': item.Attachments.Count > 0,
                    '邮件正文前200字符': item.Body[:200] if item.Body else '',
                    '邮件类别': item.Categories if item.Categories else '无'
                }
                emails_data.append(email_data)
            except Exception as e:
                print(f"处理邮件时出错: {e}")
                continue
        
        # 创建DataFrame并保存到Excel
        if emails_data:
            df = pd.DataFrame(emails_data)
            
            # 文件名包含日期
            filename = f"邮件收集_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx"
            filepath = os.path.join(os.getcwd(), filename)
            
            # 保存到Excel
            df.to_excel(filepath, index=False, engine='openpyxl')
            print(f"成功导出 {len(emails_data)} 封邮件到 {filepath}")
        else:
            print("没有找到符合条件的邮件")
            
    except Exception as e:
        print(f"导出邮件时出错: {e}")

# 定时任务设置
def schedule_export():
    # 每天9点执行
    schedule.every().day.at("09:00").do(export_emails_to_excel)
    
    # 或者每2小时执行一次
    # schedule.every(2).hours.do(export_emails_to_excel)
    
    print("邮件收集服务已启动...")
    while True:
        schedule.run_pending()
        time.sleep(60)

if __name__ == "__main__":
    # 立即执行一次
    export_emails_to_excel()
    
    # 启动定时任务（取消注释以启用）
    # schedule_export()
```

## 方案二：更详细的版本，支持更多筛选条件

```python
import win32com.client
import pandas as pd
import schedule
import time
from datetime import datetime, timedelta
import os

class OutlookEmailExporter:
    def __init__(self, output_folder="邮件导出"):
        self.output_folder = output_folder
        if not os.path.exists(output_folder):
            os.makedirs(output_folder)
    
    def get_emails(self, folder_name="收件箱", days_back=1, max_emails=1000):
        """获取指定文件夹的邮件"""
        try:
            outlook = win32com.client.Dispatch("Outlook.Application")
            namespace = outlook.GetNamespace("MAPI")
            
            # 获取指定文件夹
            if folder_name == "收件箱":
                folder = namespace.GetDefaultFolder(6)
            elif folder_name == "发件箱":
                folder = namespace.GetDefaultFolder(4)
            elif folder_name == "已发送邮件":
                folder = namespace.GetDefaultFolder(5)
            else:
                # 尝试通过名称查找文件夹
                folder = self.find_folder_by_name(namespace, folder_name)
                if not folder:
                    print(f"未找到文件夹: {folder_name}")
                    return []
            
            # 时间筛选
            since_date = datetime.now() - timedelta(days=days_back)
            restrict_items = folder.Items
            restrict_items = restrict_items.Restrict(
                f"[ReceivedTime] >= '{since_date.strftime('%m/%d/%Y')}'"
            )
            restrict_items.Sort("[ReceivedTime]", True)  # 按时间降序排列
            
            return restrict_items
        
        except Exception as e:
            print(f"获取邮件时出错: {e}")
            return []
    
    def find_folder_by_name(self, namespace, folder_name):
        """通过名称查找文件夹"""
        try:
            for folder in namespace.Folders:
                for subfolder in folder.Folders:
                    if subfolder.Name == folder_name:
                        return subfolder
                    # 递归查找子文件夹
                    found = self.search_subfolders(subfolder, folder_name)
                    if found:
                        return found
        except:
            pass
        return None
    
    def search_subfolders(self, parent_folder, target_name):
        """递归搜索子文件夹"""
        try:
            for folder in parent_folder.Folders:
                if folder.Name == target_name:
                    return folder
                found = self.search_subfolders(folder, target_name)
                if found:
                    return found
        except:
            pass
        return None
    
    def export_emails(self, folder_names=None, days_back=1):
        """导出邮件到Excel"""
        if folder_names is None:
            folder_names = ["收件箱"]
        
        all_emails_data = []
        
        for folder_name in folder_names:
            print(f"正在处理文件夹: {folder_name}")
            emails = self.get_emails(folder_name, days_back)
            
            for i, item in enumerate(emails):
                if i >= 1000:  # 限制数量
                    break
                    
                try:
                    # 提取邮件信息
                    email_data = {
                        '文件夹': folder_name,
                        '发件人邮箱': item.SenderEmailAddress if hasattr(item, 'SenderEmailAddress') else '',
                        '发件人姓名': item.SenderName if hasattr(item, 'SenderName') else '',
                        '收件人': item.To if hasattr(item, 'To') else '',
                        '主题': item.Subject if hasattr(item, 'Subject') else '',
                        '接收时间': item.ReceivedTime.strftime('%Y-%m-%d %H:%M:%S') if hasattr(item, 'ReceivedTime') else '',
                        '发送时间': item.SentOn.strftime('%Y-%m-%d %H:%M:%S') if hasattr(item, 'SentOn') else '',
                        '重要性': self.get_importance_text(item.Importance) if hasattr(item, 'Importance') else '',
                        '是否有附件': item.Attachments.Count > 0 if hasattr(item, 'Attachments') else False,
                        '附件数量': item.Attachments.Count if hasattr(item, 'Attachments') else 0,
                        '邮件大小(KB)': getattr(item, 'Size', 0) / 1024,
                        '邮件类别': item.Categories if hasattr(item, 'Categories') and item.Categories else '无',
                        '正文预览': (item.Body[:300] if hasattr(item, 'Body') and item.Body else '')
                    }
                    all_emails_data.append(email_data)
                    
                except Exception as e:
                    print(f"处理邮件时出错: {e}")
                    continue
        
        # 保存到Excel
        if all_emails_data:
            df = pd.DataFrame(all_emails_data)
            filename = f"邮件导出_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx"
            filepath = os.path.join(self.output_folder, filename)
            
            # 使用ExcelWriter来设置格式
            with pd.ExcelWriter(filepath, engine='openpyxl') as writer:
                df.to_excel(writer, sheet_name='邮件数据', index=False)
                
                # 自动调整列宽
                worksheet = writer.sheets['邮件数据']
                for column in worksheet.columns:
                    max_length = 0
                    column_letter = column[0].column_letter
                    for cell in column:
                        try:
                            if len(str(cell.value)) > max_length:
                                max_length = len(str(cell.value))
                        except:
                            pass
                    adjusted_width = min(max_length + 2, 50)
                    worksheet.column_dimensions[column_letter].width = adjusted_width
            
            print(f"成功导出 {len(all_emails_data)} 封邮件到 {filepath}")
            return filepath
        else:
            print("没有找到符合条件的邮件")
            return None
    
    def get_importance_text(self, importance_code):
        """重要性代码转文本"""
        importance_map = {
            0: "低",
            1: "普通", 
            2: "高"
        }
        return importance_map.get(importance_code, "未知")

# 使用示例
def main():
    exporter = OutlookEmailExporter()
    
    # 立即执行一次导出
    exporter.export_emails(
        folder_names=["收件箱", "已发送邮件"],  # 可以指定多个文件夹
        days_back=1  # 导出最近1天的邮件
    )
    
    # 设置定时任务（可选）
    def scheduled_export():
        print(f"{datetime.now()} 执行定时邮件导出")
        exporter.export_emails(days_back=1)
    
    # 每天上午9点执行
    schedule.every().day.at("09:00").do(scheduled_export)
    
    # 保持程序运行
    while True:
        schedule.run_pending()
        time.sleep(60)

if __name__ == "__main__":
    main()
```

## 安装所需的库

在运行脚本前，请先安装所需的Python库：

```bash
pip install pywin32 pandas schedule openpyxl
```

## 设置Windows定时任务（替代方案）

如果您不想让Python程序一直运行，可以使用Windows任务计划程序：

1. 创建一个批处理文件 `run_email_export.bat`：
```bat
@echo off
cd /d "C:\你的脚本路径"
python outlook_email_exporter.py
```

2. 在Windows任务计划程序中创建新任务：
   - 设置触发器（每天特定时间）
   - 设置操作（运行批处理文件）

## 使用说明

1. **首次运行**：先运行一次测试，确保Outlook权限正常
2. **调整参数**：根据需要修改时间范围、文件夹等参数
3. **安全考虑**：脚本只在本地运行，不会发送数据到外部
4. **性能优化**：如果邮件很多，可以调整`max_emails`参数

这个方案应该能满足您的需求，您可以根据实际情况调整代码。需要我解释任何部分或进行修改吗？

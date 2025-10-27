# GitHub Actions 部署指南

本文档说明如何在GitHub上部署自动化的arXiv论文获取脚本。

## 部署步骤

### 1. 准备GitHub仓库

确保你的项目结构如下：

```
automation/
├── .github/
│   └── workflows/
│       └── daily-arxiv.yml          # GitHub Actions工作流配置
├── get_daily_arxiv_paper.py         # 主脚本
├── docs/
│   └── daily/                       # 论文输出目录
├── requirements.txt                 # Python依赖（可选）
└── README.md
```

### 2. 设置GitHub Secrets

在GitHub仓库中添加必需的密钥：

1. 进入仓库页面
2. 点击 **Settings** → **Secrets and variables** → **Actions**
3. 点击 **New repository secret**
4. 添加以下密钥：
   - **Name**: `DEEPSEEK_API_KEY`
   - **Value**: 你的DeepSeek API密钥

### 3. 配置工作流权限

确保GitHub Actions有权限提交代码：

1. 进入 **Settings** → **Actions** → **General**
2. 滚动到 **Workflow permissions**
3. 选择 **Read and write permissions**
4. 勾选 **Allow GitHub Actions to create and approve pull requests**
5. 点击 **Save**

### 4. 创建初始目录结构

如果 `docs/daily/` 目录不存在，先创建它：

```bash
mkdir -p docs/daily
git add docs/daily/.gitkeep
git commit -m "Create docs/daily directory"
git push
```

或者创建一个 `.gitkeep` 文件：

```bash
echo "" > docs/daily/.gitkeep
```

### 5. 推送代码到GitHub

```bash
git add .
git commit -m "Add GitHub Actions workflow for daily paper fetching"
git push origin main
```

## 工作流说明

### 自动执行时间

- **定时执行**: 每天 UTC 00:00 (北京时间 08:00)
- **手动触发**: 可在 Actions 页面手动运行

### 执行流程

1. **检出代码**: 获取仓库最新代码
2. **设置Python环境**: 安装Python 3.10
3. **安装依赖**: 安装必需的Python包
4. **运行脚本**: 执行论文获取和分析
5. **提交更改**: 自动提交新生成的markdown文件

### 自定义配置

#### 修改执行时间

编辑 `.github/workflows/daily-arxiv.yml` 中的 cron 表达式：

```yaml
schedule:
  # 格式: 分 时 日 月 星期 (UTC时间)
  - cron: '0 0 * * *'    # 每天 00:00 UTC
  - cron: '0 8 * * *'    # 每天 08:00 UTC (北京时间 16:00)
  - cron: '0 */6 * * *'  # 每6小时执行一次
```

#### 修改处理日期

如果需要处理特定日期而不是昨天，修改 `get_daily_arxiv_paper.py` 的 `main()` 函数：

```python
def main():
    processor = CompletePaperProcessor()
    
    # 选项1: 处理昨天 (默认)
    target_date = (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')
    
    # 选项2: 处理今天
    # target_date = datetime.now().strftime('%Y-%m-%d')
    
    # 选项3: 处理最近7天
    # target_date = [
    #     (datetime.now() - timedelta(days=7)).strftime('%Y-%m-%d'),
    #     (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')
    # ]
    
    processor.process_papers_by_date(
        target_date=target_date,
        categories=['cs.DC', 'cs.AI', 'cs.LG'],
        max_workers=10,
        max_papers=None
    )
```

#### 调整并发数量

根据API限制调整 `max_workers` 参数：

```python
processor.process_papers_by_date(
    target_date=target_date,
    max_workers=5,  # 降低并发数，避免API限流
    max_papers=None
)
```

## 监控和调试

### 查看执行日志

1. 进入仓库的 **Actions** 标签
2. 点击最近的工作流运行记录
3. 查看详细的执行日志

### 手动触发工作流

1. 进入 **Actions** 标签
2. 选择 **Daily arXiv Paper Fetcher**
3. 点击 **Run workflow** 按钮
4. 选择分支并确认运行

### 常见问题

#### 问题1: Permission denied 错误

**解决方案**: 检查工作流权限设置（见第3步）

#### 问题2: API调用失败

**解决方案**: 
- 检查 `DEEPSEEK_API_KEY` 是否正确设置
- 检查API配额是否用完
- 降低 `max_workers` 并发数

#### 问题3: 没有提交任何更改

**原因**: 可能没有找到感兴趣的论文

**检查**: 查看Actions日志中的 "处理完成！" 输出

#### 问题4: 找不到 docs/daily 目录

**解决方案**: 确保目录已创建并提交到仓库

```bash
mkdir -p docs/daily
echo "" > docs/daily/.gitkeep
git add docs/daily/.gitkeep
git commit -m "Create docs/daily directory"
git push
```

## 高级配置

### 添加通知

可以添加Slack、Email等通知，在工作流失败时提醒：

```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'Daily paper fetch failed!'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### 使用缓存加速

工作流已配置pip缓存，如需添加其他缓存：

```yaml
- name: Cache PDFs
  uses: actions/cache@v3
  with:
    path: temp_pdfs
    key: pdf-cache-${{ hashFiles('**/get_daily_arxiv_paper.py') }}
```

### 添加依赖管理

创建 `requirements.txt` 文件：

```txt
requests>=2.31.0
PyPDF2>=3.0.0
openai>=1.0.0
tqdm>=4.66.0
```

然后修改工作流：

```yaml
- name: Install dependencies
  run: |
    pip install -r requirements.txt
```

## 成本估算

- **GitHub Actions免费额度**: 
  - 公开仓库: 无限制
  - 私有仓库: 每月2000分钟
  
- **单次执行时间**: 约5-15分钟（取决于论文数量）
- **每月使用**: 约150-450分钟（每天执行一次）

私有仓库完全可以在免费额度内使用。

## 测试部署

首次部署后建议手动触发测试：

1. 进入 **Actions** → **Daily arXiv Paper Fetcher**
2. 点击 **Run workflow**
3. 等待执行完成
4. 检查 `docs/daily/` 目录是否生成新文件
5. 查看commit历史确认自动提交成功

部署完成后，脚本将每天自动运行并更新论文列表！

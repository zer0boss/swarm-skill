# GitHub 仓库设置指南

## 手动步骤

### 1. 在GitHub上创建仓库

1. 打开 https://github.com/new
2. 仓库名称: `swarm-skill`
3. 描述: `Swarm v2.1 - 多智能体蜂群协作系统`
4. 选择 **Private** 或 **Public**
5. **不要**勾选 "Add a README file"
6. 点击 **Create repository**

### 2. 连接远程仓库并推送

创建完成后，GitHub会显示命令。或者运行以下命令（替换 `YOUR_USERNAME`）：

```bash
cd "C:\Users\Administrator\.claude\skills\swarm"

# 添加远程仓库
git remote add origin https://github.com/YOUR_USERNAME/swarm-skill.git

# 推送到GitHub
git branch -M main
git push -u origin main
```

### 3. 如果需要GitHub CLI

```powershell
# 使用winget安装
winget install --id GitHub.cli

# 或使用scoop安装
scoop install gh

# 登录
gh auth login

# 创建仓库并推送
gh repo create swarm-skill --private --source=. --push
```

## 当前状态

```
✅ 本地Git仓库已初始化
✅ 初始提交已创建 (commit: 975a12a)
⏳ 等待连接远程仓库
```

## 文件列表

| 文件 | 说明 |
|------|------|
| skill.md | 主技能文件 (v2.1) |
| upgrade-v2.1.md | v2.1升级文档 |
| SWARM_EXECUTION.md | 执行文档 |
| config_template.yaml | 配置模板 |

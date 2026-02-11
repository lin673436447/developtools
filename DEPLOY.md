# 自动部署配置指南

## 概述

本项目已配置GitHub Actions自动部署工作流。当你推送代码到 `main` 或 `master` 分支时，会自动部署到远程服务器。

## 配置步骤

### 1. 生成SSH密钥对

在本地机器上生成SSH密钥（如果还没有）：

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy
```

或者使用RSA格式：

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy
```

### 2. 将公钥添加到服务器

将生成的公钥添加到服务器的授权密钥中：

```bash
ssh-copy-id -i ~/.ssh/github_actions_deploy.pub user@your-server
```

或者手动添加：

```bash
cat ~/.ssh/github_actions_deploy.pub | ssh user@your-server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 3. 在GitHub仓库中配置Secrets

进入你的GitHub仓库 → Settings → Secrets and variables → Actions，添加以下Secrets：

| Secret名称 | 说明 | 示例 |
|------------|------|------|
| `SSH_PRIVATE_KEY` | SSH私钥内容 | `~/.ssh/github_actions_deploy` 文件的内容 |
| `SERVER_HOST` | 服务器IP或域名 | `192.168.1.100` 或 `example.com` |
| `SERVER_USER` | SSH用户名 | `root` 或 `ubuntu` |
| `DEPLOY_PATH` | 服务器上的部署路径 | `/var/www/html/dev-tools` |

### 4. 配置服务器目录权限

确保部署目录存在并有正确的权限：

```bash
# 创建部署目录
sudo mkdir -p /var/www/html/dev-tools

# 设置权限（根据你的Web服务器配置调整）
sudo chown -R www-data:www-data /var/www/html/dev-tools
sudo chmod -R 755 /var/www/html/dev-tools
```

### 5. 配置Web服务器

#### Nginx配置示例：

```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /var/www/html/dev-tools;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

#### Apache配置示例：

```apache
<VirtualHost *:80>
    ServerName your-domain.com
    DocumentRoot /var/www/html/dev-tools
    
    <Directory /var/www/html/dev-tools>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

## 触发部署

配置完成后，部署会在以下情况自动触发：

1. **自动触发**：推送代码到 `main` 或 `master` 分支
   ```bash
   git add .
   git commit -m "更新代码"
   git push origin main
   ```

2. **手动触发**：进入GitHub仓库 → Actions → Deploy to Server → Run workflow

## 查看部署状态

- 进入GitHub仓库 → Actions 标签页
- 点击工作流运行记录查看详细日志

## 故障排查

### 1. SSH连接失败

- 检查 `SSH_PRIVATE_KEY` 是否正确设置
- 确认服务器防火墙允许SSH连接（端口22）
- 验证私钥格式（应该是PEM格式）

### 2. 权限错误

- 检查服务器部署目录权限
- 确认SSH用户有写入权限

### 3. 部署后文件未更新

- 检查 `DEPLOY_PATH` 路径是否正确
- 查看GitHub Actions日志中的rsync输出

## 安全建议

1. **使用专用部署密钥**：不要重复使用个人SSH密钥
2. **限制密钥权限**：在服务器上限制部署用户的权限
3. **定期轮换密钥**：定期更换SSH密钥
4. **启用分支保护**：为main/master分支启用保护规则

## 自定义配置

### 排除特定文件

编辑 `.github/workflows/deploy.yml` 中的 rsync 命令，添加 `--exclude` 参数：

```yaml
rsync -avz --delete \
  --exclude='.git' \
  --exclude='.github' \
  --exclude='node_modules' \
  --exclude='*.log' \
  --exclude='.env' \
  ./ $SERVER_USER@$SERVER_HOST:$DEPLOY_PATH
```

### 部署前执行命令

如需在部署后执行命令（如清除缓存），添加步骤：

```yaml
- name: Post-deploy commands
  run: |
    ssh $SERVER_USER@$SERVER_HOST "cd $DEPLOY_PATH && your-command"
```

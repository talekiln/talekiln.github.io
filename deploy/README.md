# 部署到腾讯云

GitHub Pages 在国内访问不稳定，所以同一份静态文件再放一份到腾讯云服务器。每次推送 main，`.github/workflows/deploy-tencent.yml` 用 rsync 把文件同步过去。

## 一、服务器上做一次（Ubuntu）

```bash
# 1. 装 Caddy（自带 HTTPS 自动证书）
sudo apt-get update && sudo apt-get install -y caddy rsync

# 2. 建一个只用于部署的用户和站点目录
sudo adduser --disabled-password --gecos "" deploy
sudo mkdir -p /var/www/talekiln
sudo chown deploy:deploy /var/www/talekiln

# 3. 防火墙放行 80/443（腾讯云控制台“安全组”也要放行）
sudo ufw allow 80,443/tcp
```

## 二、生成部署专用 SSH 密钥（在你自己的电脑上）

```bash
ssh-keygen -t ed25519 -f talekiln_deploy -N "" -C "talekiln-site-deploy"
```

- 把 `talekiln_deploy.pub` 的内容追加到服务器 `/home/deploy/.ssh/authorized_keys`（目录权限 700，文件 600，属主 deploy）。
- 私钥 `talekiln_deploy` 只放进 GitHub Secrets，不要提交到仓库。

## 三、GitHub 仓库配置 Secrets

仓库 Settings → Secrets and variables → Actions → New repository secret：

| 名称 | 值 |
|---|---|
| `DEPLOY_HOST` | 服务器公网 IP 或域名 |
| `DEPLOY_USER` | `deploy` |
| `DEPLOY_SSH_KEY` | 私钥 `talekiln_deploy` 的全部内容 |
| `DEPLOY_PATH` | 可不填，默认 `/var/www/talekiln` |

配好后到 Actions → deploy-tencent → Run workflow 手动跑一次，以后每次推送 main 自动同步。

## 四、配置 Caddy

把 `deploy/Caddyfile.example` 改成你的域名后写入 `/etc/caddy/Caddyfile`，然后：

```bash
sudo systemctl reload caddy
```

- **有域名且已备案**：用文件里的域名那段，Caddy 自动签 HTTPS 证书。域名 A 记录指向服务器 IP。
- **备案还没下来**：大陆地域的腾讯云机器会拦截未备案域名的 80/443 访问。先用文件里注释掉的 `:80` 那段，通过 `http://服务器IP` 访问；备案通过后换成域名那段。
- 服务器如果在香港或海外地域，不需要备案，直接用域名那段。

## 五、检查

```bash
curl -I http://服务器IP/          # 或 https://你的域名/
ls /var/www/talekiln              # 应看到 index.html 和 assets/
```

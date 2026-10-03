# 部署到腾讯云

GitHub Pages 在国内访问不稳定，所以同一份静态文件再放一份到腾讯云服务器。每次推送 main，`.github/workflows/deploy-tencent.yml` 用 rsync 把文件同步过去。站点结构：`/` 是个人主页，`/talekiln/` 是剧窑作品页。

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

## 四、配置 Caddy（先用 IP 访问）

```bash
sudo nano /etc/caddy/Caddyfile   # 清空后粘贴仓库里 deploy/Caddyfile.example 的内容
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

- **备案审核期间**：用文件里的 `:80` 段，审核人员通过 `http://服务器IP/` 访问。注意是 `http://`，不是 `https://`：IP 没有证书，`https://IP` 会报错或连到别的服务。
- 腾讯云拦截的是**未备案域名**的访问，直接用 IP 访问不受影响。审核期间不要把域名解析到这台服务器。
- 443 端口这阶段用不到；如果 443 上还有别的程序（之前发现过 xray），先停掉，安全组里也可以只开 80。
- **备案通过后**：域名 A 记录指向服务器 IP，把 `:80` 段换成域名段，`systemctl reload caddy`，Caddy 自动签 HTTPS 证书。

## 五、检查

```bash
curl -I http://127.0.0.1/                 # 在服务器上，应返回 200
curl -s http://127.0.0.1/ | grep ICP      # 能看到备案号
ls /var/www/talekiln                      # 应有 index.html、talekiln/、assets/
```

在自己电脑浏览器打开 `http://服务器IP/` 是个人主页，`http://服务器IP/talekiln/` 是剧窑作品页。

打不开时按顺序排查：
1. 服务器上 `curl -I http://127.0.0.1/` 不是 200：看 `sudo journalctl -u caddy -n 50`。
2. 服务器上正常、外面打不开：腾讯云控制台“安全组”和 `sudo ufw status` 是否放行 80。
3. 页面是旧的或空的：GitHub 仓库 Actions → deploy-tencent 最近一次是否成功；Secrets 是否配了。

# skill-Github-Actions
> [GitHub Actions 持续集成服务及其交互式操作](https://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247487289&idx=1&sn=ec130e70f0a8887ebcdc11241af8d2ac&chksm=fbee4ff4cc99c6e2405c0715cd9b0b3b8893150fa9d77911a8702911bc0a5de8b78ea9b5f9df&scene=21#wechat_redirect)

GitHub Actions[1] 是 GitHub 自带的持续集成（CI）解决方案，于2018年10月正式发布[3]。它允许用户通过一系列可复用的操作（actions）构建强大的自动化工作流，这些 actions 可执行诸如抓取代码、运行测试、登录远程服务器以及部署到第三方服务等任务。每个 action 都存储在 GitHub 的代码仓库中，可以直接引用和组合使用。

GitHub Actions 提供的服务器环境配置如下：

- 2-core CPU
- 7 GB RAM 内存
- 84 GB SSD 硬盘空间

支持的操作系统包括 Ubuntu、Windows Server 2019 和 macOS X Catalina 10.15。

值得注意的是，GitHub Actions 本身并不直接支持 SSH 连接进行实时交互式操作。但本文将介绍如何巧妙绕过这一限制，实现与 GitHub Actions 服务器的连接。

**警告：**
> 使用此类方法时，请确保遵循相关服务条款，仅用于合法和正当目的。任何滥用行为导致的后果如账号封禁、资源浪费以及其他不良影响，作者概不负责。

## 方案一
mxschmitt/action-tmate[4]

首个实现在 GitHub Actions 服务器上启用 tmate 连接的动作。虽然此方案在断开连接后无法继续后续步骤，但它为 SSH 连接提供了基础功能。

**workflow 示例:**

```yaml
name: CI
on: [push]
jobs:
 build:
  runs-on: ubuntu-latest
  steps:
   - uses: actions/checkout@v2
   - name: Setup tmate session
     uses: mxschmitt/action-tmate@v2
```
## 方案二
csexton/debugger-action[6]

这个动作同样基于 tmate 实现连接，但能够在退出连接后继续执行下一步骤，更适合实际项目中的应用。该动作默认设置了一个15分钟的自动断开时间，可通过 
touch /tmp/keepalive
 命令取消。

**workflow 示例:**
```yaml
name: debugger-action
on:
 watch:
  types: started
jobs:
 build:
  runs-on: ubuntu-latest
  steps:
   - uses: actions/checkout@v2

   - name: Setup Debug Session
     uses: csexton/debugger-action@master
```

## 方案三
此方案非基于 actions 实现，而是利用 ngrok 创建 TCP 隧道穿透内网来建立 SSH 连接。
```bash
#!/bin/bash


if [[ -z "$NGROK_TOKEN" ]]; then
  echo "Please set 'NGROK_TOKEN'"
  exit 2
fi

if [[ -z "$USER_PASS" ]]; then
  echo "Please set 'USER_PASS' for user: $USER"
  exit 3
fi

echo "### Install ngrok ###"

wget -q https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-386.zip
unzip ngrok-stable-linux-386.zip
chmod +x ./ngrok

echo "### Update user: $USER password ###"
echo -e "$USER_PASS\n$USER_PASS" | sudo passwd "$USER"

echo "### Start ngrok proxy for 22 port ###"


rm -f .ngrok.log
./ngrok authtoken "$NGROK_TOKEN"
./ngrok tcp 22 --log ".ngrok.log" &

sleep 10
HAS_ERRORS=$(grep "command failed" < .ngrok.log)

if [[ -z "$HAS_ERRORS" ]]; then
  echo ""
  echo "=========================================="
  echo "To connect: $(grep -o -E "tcp://(.+)" < .ngrok.log | sed "s/tcp:\/\//ssh $USER@/" | sed "s/:/ -p /")"
  echo "=========================================="
else
  echo "$HAS_ERRORS"
  exit 4
fi
```

首先需要在 ngrok 的官网[8] 注册一个账户，并生成一个Tunnel Authtoken：https://dashboard.ngrok.com/auth
。然后在 workflow 中配置如下：
```yaml
name: Debugging with SSH
on: push
jobs:
 build:
  runs-on: ubuntu-latest
  steps:
   - uses: actions/checkout@v1

   - name: Try Build
     run: ./not-exist-file.sh it bloke build

   - name: Start SSH via Ngrok
     if: ${{ failure() }}
     run: curl -sL https://gist.githubusercontent.com/retyui/7115bb6acf151351a143ec8f96a7c561/raw/7099b9db76729dc5761da72aa8525f632d8875c9/debug-github-actions.sh | bash
     env:
      # After sign up on the https://ngrok.com/
      # You can find this token here: https://dashboard.ngrok.com/get-started/setup
      NGROK_TOKEN: ${{ secrets.NGROK_TOKEN }}

      # This password you will use when authorizing via SSH 
      USER_PASS: ${{ secrets.USER_PASS }}

   - name: Don't kill instace
     if: ${{ failure() }}
     run: sleep 1h # Prevent to killing instance after failure
```
在此方案中，**NGROK_TOKEN** 和 **USER_PASS** 应当从 GitHub Secrets 中安全获取。

**最后提醒**： 请广大开发者以学习研究为目的使用以上方法，切勿滥用或用于恶意用途。

**参考资料**：
- [1] [GitHub Actions](https://github.com/features/actions))
- [2] [持续集成服务](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)
- [3] [推出](https://github.blog/changelog/2018-10-16-github-actions-limited-beta/)
- [4] [mxschmitt/action-tmate](https://p3terx.com/go/aHR0cHM6Ly9naXRodWIuY29tL214c2NobWl0dC9hY3Rpb24tdG1hdGU=)
- [5] [tmate](https://github.com/tmate-io/tmate)
- [6] [csexton/debugger-action](https://p3terx.com/go/aHR0cHM6Ly9naXRodWIuY29tL2NzZXh0b24vZGVidWdnZXItYWN0aW9u)
- [7] [mxschmitt/action-tmate](https://p3terx.com/go/aHR0cHM6Ly9naXRodWIuY29tL214c2NobWl0dC9hY3Rpb24tdG1hdGU=)
- [8] [ngrok 的官网](https://ngrok.com/)
- [9] [官方文档](https://docs.github.com/cn/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)
- [10] [SSH 连接到 GitHub Actions 虚拟服务器环境](https://p3terx.com/archives/ssh-to-the-github-actions-virtual-server-environment.html)
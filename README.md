# Turn-Windows-Into-Server
A step-by-step guide to turning a Windows 10/11 machine into an SSH server with VSCode remote access support.
🧠 Windows 本地账户 + SSH 远程连接配置教程（适用于 Win10 / Win11）

---

## 🧑‍💻 一、设置本地账户

1. 打开 `设置` → `账户` → `您的信息` → 点击 **改为使用本地账户登录**
2. 设置本地账户用户名和密码，例如：`MikeMengTR`
   > ✅ 这是用于后续 SSH 登录的用户名和密码，与 Microsoft 帐号无关！
3. 下次登录 Windows 时使用本地账户名和密码
4. 查看当前用户名的方法：
   ```powershell
   whoami  # 返回格式：电脑名\用户名
   ```
   > 其中 `\` 后面的即是你的本地用户名

---

## 🔧 二、安装 & 配置 OpenSSH

### 2.1 卸载旧版本（如果之前有安装过）

- 卸载已安装的 OpenSSH 客户端/服务器
- 删除目录：
  ```
  C:\Users\用户名\.ssh
  ```
  > ❗ 如果你没有安装过，可以跳过这步；如果你之前尝试配置失败，一定要删除该目录重新来过

### 2.2 安装 OpenSSH 客户端/服务器

- 打开 `设置` → `应用` → `可选功能`
- 添加功能：**OpenSSH Client** 和 **OpenSSH Server**

### 2.3 启动 ssh 服务并设置开机自启

```powershell
Set-Service -Name sshd -StartupType 'Automatic'  # 设置为开机自启
Start-Service sshd                               # 启动 sshd 服务
Get-Service sshd                                 # 查看状态是否为 Running
```

---

## 🔥 三、配置防火墙放行 SSH 端口（22）

### 方法一：图形界面配置

1. 打开 `控制面板` → `系统和安全` → `Windows Defender 防火墙` → `高级设置`
2. 点击【入站规则】→【新建规则】
3. 选择类型为【端口】
4. 选择协议为 TCP，特定端口填 `22`
5. 选择【允许连接】
6. 勾选【域】【专用】【公用】全部选项
7. 命名规则为 `SSH`

### 方法二：PowerShell 命令（推荐）

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

### 方法三：兼容旧系统命令

```powershell
netsh advfirewall firewall add rule name="Allow SSH In" ^
  protocol=TCP dir=in localport=22 action=allow
```

---

## 🌐 四、启用 ping（ICMP）功能（可选但推荐）

若客户端无法 ping 通服务端 IP，可能是因为服务端防火墙屏蔽了 ICMP。在服务端中用管理员权限的 PowerShell／CMD 一次性添加规则：

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4 In" ^
  protocol=icmpv4:8,any dir=in action=allow
```

### 验证 ICMP：

```powershell
ping 127.0.0.1  # 在服务端中运行验证本地回环 ping
tracert 服务端IP  # 在客户端中运行查看网络路径是否能到达服务端
ping 服务端IP     # 在客户端中运行再次测试能否 ping 通
```

---

## 🔑 五、配置 SSH 密钥认证（可选，但更安全）

### 5.1 客户端生成 SSH 密钥对

```powershell
ssh-keygen
# 按回车一路默认生成到：C:\Users\你的用户名\.ssh\id_rsa（私钥）与 id_rsa.pub（公钥）
```

### 5.2 服务端配置公钥

- 将 `id_rsa.pub` 文件内容复制到服务端 `C:\Users\你的用户名\.ssh\authorized_keys`
  > ✅ 如果 `.ssh` 或 `authorized_keys` 文件不存在，请手动创建
  > ⚠ 注意：authorized\_keys **不要有后缀名**，必须无扩展名，且需管理员权限创建
  >
  > 🔒 建议为 `.ssh` 文件夹和 `authorized_keys` 设置权限，仅保留当前用户与 SYSTEM 的所有访问权限，移除其他用户权限

### 5.3 修改 sshd 配置文件以启用密钥认证

- 打开文件：`C:\ProgramData\ssh\sshd_config`
- 修改以下内容（确保去掉注释符号 `#`）：
  ```
  PubkeyAuthentication yes         # 启用密钥认证
  PasswordAuthentication yes       # 允许密码登录（可选）
  AuthorizedKeysFile .ssh/authorized_keys
  ```

### 5.4 重启 SSH 服务

```powershell
Restart-Service sshd
```

---

## 💻 六、客户端测试 SSH 连接

### 命令行连接方式（密码登录或密钥登录）

```bash
# 密码登录
ssh 用户名@服务端IP

# 密钥登录（路径为私钥路径）
ssh -i ~/.ssh/id_rsa 用户名@服务端IP
```

> ❗ 不论使用密码还是密钥登录，命令格式都是 `ssh 用户名@服务端IP`，仅密钥登录多了 `-i` 参数

---

## 🖥️ 七、使用 VSCode Remote-SSH 插件连接

1. 打开 VSCode，安装插件 `Remote - SSH`

2. Ctrl + Shift + P → 输入并选择 `Remote-SSH: Add New SSH Host`

3. 选择 `C:\Users\你的用户名\.ssh\config` 配置文件

4. 添加配置内容如下：

   ```
   Host my-windows-laptop            # ← 这里你可以自己起名字
       HostName xxx.xxx.xxx.xxx      # ← 这里填上你的 Windows 笔记本的IP（服务端IPv4地址）
       User MikeMengTR               # ← 这里填上你的 Windows 用户名
       Port 22                       # ← 默认端口就是22，但最好写上
       # IdentityFile ~/.ssh/id_rsa  # ← 如果用密钥认证可以加这一行
   ```

5. 保存后在左侧远程资源管理器中点击主机名连接

   > 若使用密码连接，系统会提示输入密码；若配置了密钥，将自动连接

---

## ✅ 总结 Checklist
 ```
  \- [x] 使用本地账户登录系统

  \- [x] 安装并配置 OpenSSH 客户端与服务端

  \- [x] 启用防火墙规则放行 22 端口

  \- [x] （可选）启用 ICMP Ping 功能

  \- [x] 配置 SSH 密钥认证

  \- [x] 成功连接并使用 VSCode Remote-SSH
 ```

🚀 你现在应该能够顺利从客户端通过 SSH 或 VSCode 实现远程连接和控制你的 Windows 服务器了！


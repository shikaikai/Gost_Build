# Gost_Build

## 项目说明

需求是在服务上部署一个路由,转发通过这个路由的所有请求。
简单理解就是代理转发(支持负载均衡)

> 首先感谢大佬们的无私开发
> 原文链接：https://github.com/ginuerzh/gost
> 详细部署说明：https://v2.gost.run/getting-started/
> 这里只是一个部署说明，只实现了我要的需求，仅供小白参考！！！

---

### 部署

复杂型种的简单型，可以自由指定代理链中节点组中节点的类型。
![Gost_Proxy](./gost_proxy.png)

可以下载源码自己编译修改，也可以直接用二进制文件部署
这里简单介绍下二进制文件部署吧
官方下载目录 https://github.com/ginuerzh/gost/releases

1. **安装 gost**
   首先，从 GitHub 下载并安装 gost。假设你正在使用 64 位的 Linux 系统(可以去官方目录里找适合自己版本的)：

   ```bash
    wget https://github.com/ginuerzh/gost/releases/download/v2.11.1/gost-linux-amd64-2.11.1.gz
    gunzip gost-linux-amd64-2.11.1.gz
    chmod +x gost-linux-amd64-2.11.1
    sudo mv gost-linux-amd64-2.11.1 /usr/local/bin/gost
   ```

2. **创建配置文件**
   在服务器上创建一个配置文件来定义代理规则。创建一个名为 gost.json 的文件，并将其放在合适的位置，例如/etc/gost/gost.json。
   ```bash
   mkdir -p /etc/gost
   cd /etc/gost
   vim gost.json
   ```
3. **配置 gost.json**

   ```json
   {
     "Debug": true,
     "Retries": 3,
     "Routes": [
       {
         "Retries": 3,
         "ServeNodes": ["admin:admin@0.0.0.0:8888"],
         "ChainNodes": [":1080?peer=peer.txt"]
       }
     ]
   }
   ```

参数说明

- **Debug**:
- 类型: `boolean`
- 说明: 启用或禁用调试模式。如果设置为 `true`，将记录详细的调试信息，方便排查问题。

- **Retries**:
- 类型: `integer`
- 说明: 请求失败时的重试次数。如果请求超出此次数仍然失败，将停止尝试。

- **Routes**:
- 类型: `array`
- 说明: 配置路由的数组，每个路由包含以下参数：

- **Retries**:

  - 类型: `integer`
  - 说明: 特定路由请求失败时的重试次数。超出此次数将不再重试。

- **ServeNodes**:

  - 类型: `array`
  - 说明: 服务器节点的地址和端口配置。此处配置为 `admin:admin@0.0.0.0:8888`，表示服务将在此地址和端口上监听请求,并配置了认证信息

- **ChainNodes**:

  - 类型: `array`
  - 说明: 代理链节点的地址和端口，可以指定多个节点，`?peer=peer.txt` 用于额外配置或参数。

> 这个配置用于设置一个带有身份验证的代理服务器，并实现请求的重试机制。不想要身份认证可以去掉，直接`0.0.0.0:8888`也是 ok 的
> 详情参考：[节点配置](https://v2.gost.run/configuration/)

4. **创建并配置 peer.txt**

   ```bash
   vim peer.txt
   ```

   写入

   ```txt
   strategy        round
   max_fails       1
   fail_timeout    30s
   reload          10s

   # peers
   peer    socks5://user1:pass1@ip1:port1
   peer    socks5://user2:pass2@ip2:port2
   peer    socks5://user3:pass3@ip3:port3

   ```

   参数说明

   - **strategy**:
   - 说明: 节点选择策略，用于确定请求时选择哪个节点。可选值包括 `round`（轮询），`random`（随机），或 `fifo`（先进先出）。这里配置为 `round`，表示采用轮询方式选择节点。

   - **max_fails**:
   - 说明: 最大失败次数，超过此次数后，节点将被标记为死亡节点，不再参与请求。这里设置为 `1`，即请求失败一次后，该节点将被标记为死亡。

   - **fail_timeout**:
   - 说明: 节点被标记为死亡后的超时时间，超出此时间后，节点将重新参与请求选择。这里设置为 `30s`，表示节点在 30 秒后会恢复可用状态。

   - **reload**:
   - 说明: 配置文件热更新的检查周期，表示在此时间间隔内检查配置文件是否有更新。这里设置为 `10s`，即每 10 秒检查一次配置文件的更新。

   - **peers**:
   - 说明: SOCKS5 代理节点的具体地址列表。每个节点都包含认证信息，用于安全访问。

   - **peer**:
     - 类型: `string`
     - 说明: 具体的节点地址，格式为 `socks5://username:password@ip:port`，例如：
     - `socks5://user1:pass1@ip1:port1`
     - `socks5://user2:pass2@ip2:port2`
     - `socks5://user3:pass3@ip3:port3`
     - 这些配置为可用的 SOCKS5 代理节点，支持身份验证。当然也支持 socks5://ip:port

   此配置用于设置一个基于 SOCKS5 的代理系统，能够选择合适的节点进行请求，并具有失败处理和动态更新的能力。

#### 使用方法(直接调用 -- 存在进程意外丢失的问题，推荐另一个方案，在最下面)

将上述两个配置文件 gost.json 和 peer.txt 保存在服务器上，启动 GOST 时使用以下命令：

```bash
gost -C gost.json > gost.log 2>&1 &
```

这样，GOST 将会根据配置文件中的设置，通过轮询策略在 peer.txt 中指定的节点之间进行负载均衡，确保每次请求都从其中一个节点出去。

---

### 测试

```bash
for i in {1..10}; do curl -x socks5://admin:admin@localhost:8888 http://ifconfig.me; echo ""; done
```

对了如果要让客户端能正常访问要暴露出 8888 端口(配置防火墙)，并且用你服务的 ip 地址替换掉 localhost 再测试一次。

```bash
#开放服务器8888端口
sudo firewall-cmd --permanent --add-port=8888/tcp
#添加完端口后，重新加载防火墙规则以使更改生效
sudo firewall-cmd --reload
#验证端口是否已开放，运行以下命令以确认 8888 端口已成功添加
sudo firewall-cmd --list-all
#在输出中看到 ports 行中包含 8888/tcp。

#确保服务正在监听 8888 端口
sudo netstat -tuln | grep 8888

#检查程序是否正常运行
curl -x socks5h://admin:admin@server_ip:8888 http://ifconfig.me

```

**_ok！ 整个过程部署完毕，我相信小白也一定部署成功了，再次感谢大佬们的贡献。_**

---

### [补充]

为了要确保 `gost` 进程持续运行并能在崩溃后自动重启，可以使用 `systemd` 来创建一个服务。以下是创建 `gost` 服务的步骤：

1. **创建服务文件**：
   在 `/etc/systemd/system/` 目录下创建一个名为 `gost.service` 的文件。

   ```bash
   sudo nano /etc/systemd/system/gost.service
   ```

   在文件中添加以下内容（根据你的实际路径和配置进行修改）：

   ```ini
   [Unit]
   Description=gost service
   After=network.target

   [Service]
   ExecStart=/usr/local/bin/gost -C /path/to/gost.json
   Restart=always
   RestartSec=3
   StandardOutput=file:/path/to/gost.log
   StandardError=inherit

   [Install]
   WantedBy=multi-user.target
   ```

   注意：

   - 将 `/usr/local/bin/gost` 替换为 `gost` 的实际安装路径。
   - 将 `/path/to/gost.json` 替换为你的配置文件的实际路径。
   - `Restart=always` 确保服务崩溃后自动重启。
   - `RestartSec=3` 设置重启前的延迟时间。

   **修改配置 gost.json**

   [将 peer.txt 的绝对路径写上]

   ```json
   {
     "Debug": true,
     "Retries": 3,
     "Routes": [
       {
         "Retries": 3,
         "ServeNodes": ["admin:admin@0.0.0.0:8888"],
         "ChainNodes": [":1080?peer=/etc/gost/peer.txt"]
       }
     ]
   }
   ```

2. **重新加载 `systemd` 管理器配置**：

   ```bash
   sudo systemctl daemon-reload
   ```

3. **启动 `gost` 服务**：

   ```bash
   sudo systemctl start gost
   ```

4. **设置开机自动启动**：

   ```bash
   sudo systemctl enable gost
   ```

5. **检查服务状态**：

   ```bash
   sudo systemctl status gost
   ```

通过以上步骤，可以确保 `gost` 作为守护进程运行，并在意外退出后自动重启。如果你需要查看日志，可以直接查看指定的 `gost.log` 文件。

**项目名称：**  GEACX2 主从机网络共享与代理环境部署

**适用平台：**  GEACX2 (Master/Slave), Ubuntu 20.04 (L4T Kernel)

**文档版本：**  V1.0

**生成日期：**  2026年1月28日

---

## 1. 背景与目标

GEACX2 平台内部通过 PCIE 虚拟化网口 (`eth10`) 进行高速通信。默认情况下，从机 (Slave) 无法访问外部网络，导致无法进行 `git clone`、`pip install` 或时间同步。

**本配置方案旨在实现以下目标：**

1. **基础互联：**  建立主从机之间稳定的静态 IP 路由。
2. **网络共享：**  主机 (Master) 作为网关，将 WiFi 网络共享给从机，实现从机直连上网。
3. **按需代理：**  在不干扰底层 ROS 通信的前提下，从机可通过指令一键开启代理服务（复用主机的 Clash），解决开发环境依赖下载问题。

---

## 2. 网络拓扑与规划

| **节点** | **接口名称** | **IP 地址**               | **角色**                  | **备注**                           |  |  |  |  |  |
| -- | -- | ---------------- | ------------------- | ---------------------------- | -- | -- | -- | -- | -- |
| **主机 (Master)** | `wlan0` | DHCP (外网 IP) | 网关 / 代理服务器 | 负责 NAT 转发与 Clash 托管 |  |  |  |  |  |
| **主机 (Master)** | `eth10` | `192.168.44.100`               | 内部网关          | PCIE 虚拟网口              |  |  |  |  |  |
| **从机 (Slave)** | `eth10` | `192.168.44.101`               | 客户端            | 网关指向 192.168.44.100    |  |  |  |  |  |

---

## 3. 详细实施步骤

### 阶段一：从机 (Slave) 静态 IP 与路由配置

**目标：**  固定从机 IP 并指定主机为默认网关。

**操作对象：**  从机终端

1. 编辑 Netplan 配置文件：
    Bash

    ```
    sudo vi /etc/netplan/01-network-manager-all.yaml
    ```
2. **严格按照以下格式修改**（注意缩进必须使用空格，严禁 Tab）：
    YAML

    ```
    network:
      version: 2
      renderer: NetworkManager
      ethernets:
        eth10:
          dhcp4: no
          addresses:
            - 192.168.44.101/24
          routes:
            - to: default
              via: 192.168.44.100
          nameservers:
            addresses: [114.114.114.114, 8.8.8.8]
    ```
3. 应用配置：
    Bash

    ```
    sudo netplan apply
    ```

### 阶段二：主机 (Master) NAT 转发配置

**目标：**  开启内核转发功能，并设置防火墙规则，允许从机流量通过。

**操作对象：**  主机终端

1. **永久开启内核转发：** 
    Bash

    ```
    echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```
2. **安装规则持久化工具：** 
    Bash

    ```
    sudo apt update
    sudo apt install iptables-persistent
    # 安装过程中弹出的确认框均选 "Yes"
    ```
3. **配置 NAT 规则（假设主机使用 wlan0 上网）：** 
    Bash

    ```
    # 清理旧规则
    sudo iptables -t nat -F POSTROUTING
    # 添加 SNAT 伪装规则
    sudo iptables -t nat -A POSTROUTING -s 192.168.44.0/24 -o wlan0 -j MASQUERADE
    # 允许转发链路
    sudo iptables -A FORWARD -i eth10 -o wlan0 -j ACCEPT
    sudo iptables -A FORWARD -i wlan0 -o eth10 -m state --state RELATED,ESTABLISHED -j ACCEPT
    # 保存规则
    sudo netfilter-persistent save
    ```

### 阶段三：主机 (Master) 代理服务配置

**目标：**  配置 Clash 允许局域网连接，使从机可以“借用”主机的代理。

**操作对象：**  主机 Clash 配置文件

1. 修改 Clash 的 `config.yaml` 或在 GUI 界面设置：

    - **Allow LAN (允许局域网):**  `true` (开启)
    - **Port (端口):**  `7890` (默认混合端口)
2. **验证监听状态：** 
    在主机执行 `netstat -an | grep 7890`，确认监听地址为 `:::` 或 `0.0.0.0`，而非仅 `127.0.0.1`。

### 阶段四：从机 (Slave) 客户端脚本配置

**目标：**  设置快捷指令，实现“默认直连，按需代理”。

**操作对象：**  从机终端

1. 编辑用户环境变量文件：
    Bash

    ```
    nano ~/.bashrc
    ```
2. 在文件末尾添加以下别名：
    Bash

    ```
    # === 代理快捷指令 ===
    # 开启代理 (指向主机 192.168.44.100)
    alias proxy='export http_proxy="http://192.168.44.100:7890"; export https_proxy="http://192.168.44.100:7890"; export all_proxy="socks5://192.168.44.100:7890"; echo "Proxy ON (via Master)"'

    # 关闭代理 (恢复直连)
    alias unproxy='unset http_proxy; unset https_proxy; unset all_proxy; echo "Proxy OFF (Direct)"'
    ```
3. 刷新生效：
    Bash

    ```
    source ~/.bashrc
    ```

---

## 4. 使用与验证指南

### 4.1 验证基础网络（直连模式）

- **操作：**  确保终端未开启代理（输入 `unproxy`）。
- **测试：**  `ping www.baidu.com`
- **预期：**  能 Ping 通，延迟正常（约 10-50ms）。此模式适用于 ROS 通信、相机数据传输。

### 4.2 验证代理网络（开发模式）

- **操作：**  输入指令 `proxy`。
- **测试：**  `curl -I https://www.google.com`
- **预期：**  返回 `HTTP/2 200` 状态码。此模式适用于 `git clone`, `apt install`, `pip install`。

### 4.3 浏览器使用代理

- **方法：**  在从机终端输入 `chromium-browser --proxy-server="http://192.168.44.100:7890"` 启动浏览器。

---

## 5. 常见问题与避坑 (FAQ)

**Q1:**  **`sudo netplan apply`** **报错** **`Invalid YAML`** **或** **`expected sequence`** **？**

- **原因：**  缩进错误或格式符号错误。
- **解决：**  确保使用**空格**而非 Tab 键进行缩进；确保 `-` 和 `to` 之间有空格；确保 `eth10` 与 `eth0` 对齐。

**Q2: 从机能 Ping 通 IP，但 Ping 不通域名（DNS 错误）？**

- **原因：**  Netplan 中的 `nameservers` 未生效或主机 NAT 转发异常。
- **解决：**  检查从机 `/etc/resolv.conf` 是否包含 `114.114.114.114`。确认主机已执行 `iptables` 配置。

**Q3: 为何不直接在从机运行 v2rayA/Clash 的透明代理？**

- **原因：**  GEACX2 使用定制的 Tegra 内核，裁剪了 `xt_TPROXY` 等必要模块，导致透明代理无法启动。且双重透明代理易引发路由死循环。
- **策略：**  采用本文档的“网络层直连 + 应用层代理”的分离策略是最稳定的方案。

---

## 6. 安全警示 (Critical Warnings)

1. **禁止执行** **`apt upgrade`** **：**  严禁在 GEACX2 上执行全系统升级，这会导致 NVIDIA 定制驱动（包括 PCIE 网卡驱动）失效，导致 `eth10` 消失，主从通信中断。仅可使用 `apt update` 和 `apt install <包名>`。
2. **IP 地址冲突：**  确保主机连接的 WiFi 网段不要是 `192.168.44.x`，否则会导致路由冲突。

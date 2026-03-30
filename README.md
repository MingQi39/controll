# Home Assistant + 米家餐厅灯 + Bose 蓝牙音箱（本地 Docker）

在家里的电脑上按顺序做即可。本文假设你有一台**常开**的电脑或树莓派，能装 Docker，且音箱通过**蓝牙**连在这台机器上。

---

## 你要实现什么

| 项目 | 说明 |
|------|------|
| 触发 | 米家 App 里把「餐厅灯」打开（或任何会点亮该灯的操作） |
| 动作 | 本机已配对的 Bose SoundLink Mini II 播放一段音乐 |
| 运行 | Home Assistant 用 Docker 常驻，配置保存在本地文件夹 |

---

## 整体流程（一眼看懂）

```text
米家 App（手机）
       ↓
米家设备（餐厅灯）状态变为「开」
       ↓
Home Assistant（Docker）检测到灯实体变为 on
       ↓
自动化 → 调用脚本 → shell_command 在宿主机播放音频
       ↓
蓝牙已连接的 Bose 音箱出声
```

```mermaid
flowchart LR
  A[米家 App] --> B[餐厅灯 ON]
  B --> C[Home Assistant]
  C --> D[shell_command / aplay]
  D --> E[Bose 蓝牙音箱]
```

---

## 你需要准备什么

1. **Docker**（桌面版或 Linux 上的 Docker Engine 均可）。
2. **Home Assistant 配置目录**：本仓库里用 `homeassistant-config` 文件夹映射到容器内 `/config`（路径可按你电脑修改）。
3. **米家餐厅灯**能被 Home Assistant 识别（见下文「方式 A / B」）。
4. **蓝牙音箱**已与**运行 Docker 的那台电脑**配对并可正常 `aplay` 测试。

---

## 第一步：安装并启动 Home Assistant（Docker）

### 方式 1：本仓库提供的 Docker Compose（推荐）

1. 把整个项目文件夹拷到你家里的电脑。
2. 用文本编辑器打开 `docker-compose.yml`，确认这一行里的路径是你想要的配置目录：
   - `./homeassistant-config:/config`
   - Windows 可改成例如 `D:/homeassistant/config:/config`（注意 Docker Desktop 里要共享该盘符）。
3. 在项目根目录执行：

```bash
docker compose up -d
```

4. 浏览器打开：`http://你电脑的局域网IP:8123`（同一 WiFi 下的手机也可以访问这个地址做初始设置）。
5. 按向导创建管理员账号。

### 方式 2：单条 docker run（与上文二选一）

```bash
docker run -d \
  --name homeassistant \
  --restart=unless-stopped \
  -p 8123:8123 \
  -v /你的路径/homeassistant/config:/config \
  ghcr.io/home-assistant/home-assistant:stable
```

把 `/你的路径/homeassistant/config` 换成你**真实存在**的文件夹（可先手动建空文件夹）。

---

## 第二步：蓝牙音箱在宿主机上配对并测试

在**运行 Docker 的那台机器**上操作（不是手机）：

```bash
bluetoothctl
# 进入后依次：
power on
agent on
default-agent
scan on
# 看到 Bose 的 MAC 后：
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
quit
```

把测试音频放到 Home Assistant 配置目录里，示例统一命名为 **`test.wav`**（`aplay` 对 WAV 最省事）。若坚持用 MP3，需把 `shell_command` 改成 `ffplay`/`mpg123` 等，并保证容器或宿主机已安装对应命令。

```bash
aplay /你的配置目录路径/test.wav
```

若音箱能出声，说明**蓝牙 + 音频输出**在宿主机上没问题。

### 蓝牙、Docker 与 Home Assistant 的重要说明

- 默认情况下，**容器里的 HA 调用的 shell 若在容器内执行**，可能**没有**宿主机的蓝牙与声卡。
- 常见做法之一：用 `shell_command` 执行**宿主机上的脚本**（例如通过 `docker exec` 不行时要改为在宿主机定时任务或 SSH），更简单的是：
  - **使用 `network_mode: host` 且把音频设备映射进容器**（Linux 上较常见），或  
  - **在宿主机放一个脚本**（如 `/usr/local/bin/play-bose.sh`），内容只有 `aplay /path/to/test.wav`，然后在 `configuration.yaml` 里写绝对路径调用该脚本（需保证 HA 进程有权限执行）。

**新手最省事路径**：在树莓派或 Linux 宿主机上，查阅 Home Assistant 官方文档 **「Supervised / OS」** 与 **「Audio / Bluetooth」**；若仅用 Docker，优先查 **「Docker 蓝牙 passthrough」** 针对你的系统版本。

本仓库示例使用 `aplay /config/test.wav` 表示文件放在配置目录；若实际执行环境是宿主机，请把 `shell_command` 改成宿主机真实路径或脚本。

---

## 第三步：把播放命令写进 Home Assistant

1. 进入配置目录（映射到 `/config` 的那个文件夹）。
2. 参考 `homeassistant-config/examples/configuration.snippet.yaml`，把 `shell_command` 和 `script` 合并进 `configuration.yaml`。
3. 在 HA 网页：**开发者工具 → 服务**，服务选 `script.play_music`，点「调用服务」测试是否能播。

若报错，打开 **设置 → 系统 → 日志** 查看 `shell_command` 是否找不到命令或路径。

---

## 第四步：把米家餐厅灯接入 Home Assistant

### 方式 A：局域网 + Token（经典写法）

1. 查灯的局域网 IP、32 位 **token**（网上有「米家 token 提取」教程，需按你灯型号操作）。
2. 将 `xiaomi_miio` 段落合并进 `configuration.yaml`（见 `examples/configuration.snippet.yaml`），**host / token 改成你的**。
3. **重启** Home Assistant。
4. 在 **设置 → 设备与服务 → 实体**，找到灯，记下 **实体 ID**（例如 `light.can_ting_deng`），下一步自动化要用。

> 说明：新版 Home Assistant 更推荐在 **「设置 → 设备与服务 → 添加集成」** 里搜索 **Xiaomi** / **Xiaomi Miio** 用界面配置；若集成方式有变化，以你当前 HA 版本界面为准，原理仍是「有一个灯的实体，状态 on/off」。

### 方式 B：米家云（账号同步设备）

1. 在 HA 里安装 **HACS**（可选）或直接使用官方/社区 **小米云** 相关集成。
2. 登录米家账号，勾选设备同步。
3. 同样在实体列表里确认餐厅灯对应的 **entity_id**。

---

## 第五步：自动化——灯开就播放

1. 打开 `homeassistant-config/examples/automations.snippet.yaml`。
2. 若你的 `configuration.yaml` 中有：

```yaml
automation: !include automations.yaml
```

则把 `automations.yaml` 里写成**列表**（文件里已用 `- alias:` 开头，不要在外面再包一层 `automation:`）。

3. 把示例里的 `light.dining_room` 改成你在上一步记下的**真实实体 ID**。
4. 保存后，在 **开发者工具 → YAML → 重载自动化**，或重启 HA。
5. 用手机 **米家 App 开灯**，观察是否自动播放。

可选增强见 `examples/automation-with-condition.snippet.yaml`（延迟、条件）。若没有 `media_player.bose`，请删掉条件段，避免自动化永远不触发。

---

## 第六步：手机怎么用

| 方式 | 做什么 |
|------|--------|
| 米家 App | 正常开关餐厅灯 → 触发 HA 自动化 → 音箱播放 |
| Home Assistant App（可选） | 同一局域网访问 `http://IP:8123`，可看实体、手动运行脚本 |

---

## 常驻与稳定建议

- 容器已使用 `restart: unless-stopped`，宿主机重启后一般会自动拉起。
- 给运行 HA 的机器设**固定局域网 IP**，避免手机书签失效。
- 外网访问需另行配置 VPN、内网穿透或 Nabu Casa，本文不展开。

---

## 练习题（巩固）

- 复制一份自动化，改成「卧室灯开 → 播放 + 控制另一个设备（如智能插座）」。
- 一个 `action` 里写多个 `service` 调用，实现多设备联动。

---

## 思考题（进阶）

- 小爱音箱 / 语音场景：需米家或 HA 支持的场景与语音入口配合。
- 「用餐 / 睡眠 / 娱乐」模式：用 HA **场景（scene）** 或 **脚本** 批量设灯、媒体、窗帘。

---

## 仓库里有什么

| 路径 | 含义 |
|------|------|
| `docker-compose.yml` | 一键启动 HA 的 Compose 文件 |
| `homeassistant-config/` | 映射到容器 `/config` 的配置根目录 |
| `homeassistant-config/examples/` | 可复制合并的 YAML 片段 |
| `examples/automations.yaml.example` | 可改名为 `automations.yaml` 的完整列表示例（需配合 `configuration.yaml` 里的 `!include`） |
| `README.md` | 本说明 |

**说明**：本仓库是**步骤说明 + 配置片段**，不包含真实 token、也不替你跑 Docker。第一次启动 HA 后，`configuration.yaml` 可能由向导生成，请**把 `examples` 里的段落合并进去**，不要无脑覆盖整文件。

---

## 常见问题（FAQ）

**Q：灯在米家 App 能开关，HA 里没实体？**  
A：检查集成是否添加成功、IP/token 是否正确，或尝试用 UI 集成替代手写 YAML。

**Q：自动化触发但没声音？**  
A：先确认在**同一环境**（宿主机或容器内）手动执行 `shell_command` 里的命令能播；再查路径是 `/config/...` 还是宿主机绝对路径。

**Q：`xiaomi_miio` 报错？**  
A：不同 HA 版本集成名称可能不同，以官方文档与「添加集成」里的选项为准。

---

## 架构图（可选）

若你需要**更花哨的一张联动示意图**（适合打印或放在备忘录里），可以在本地用画图工具照着本文开头的流程图临摹，或说明你的环境（Windows / 树莓派 / NAS）再单独画一版设备连线图。

---

祝你在家里一次跑通：米家开灯 → Home Assistant → Bose 出声。

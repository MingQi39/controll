# 这是 Home Assistant 配置目录占位说明

把本文件夹当作 Docker 挂载到容器里的 `/config`。

**在家里的电脑上：**

1. 把 `examples/` 里的内容按需合并进 `configuration.yaml`；自动化可复制 `automations.yaml.example` 为 `automations.yaml`（并确保 `configuration.yaml` 含 `automation: !include automations.yaml`）。仅遥控 TCL 电视时见仓库根目录 `README-TCL-TV.md` 与 `examples/tcl_tv_scripts.snippet.yaml`。
2. 把测试音频命名为 `test.wav` 放在本目录（与示例里 `aplay /config/test.wav` 对应；`aplay` 对 MP3 支持因系统而异）。
3. 首次启动 HA 后，部分文件可能由系统自动生成；若与示例冲突，以 README 的合并步骤为准。

删除本说明文件不影响运行。

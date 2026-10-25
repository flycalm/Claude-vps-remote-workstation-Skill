# Claude VPS Remote Workstation

低配 Linux VPS 海外远程桌面工作站搭建与 Claude 登录风控排障经验。

## 阅读指南

**[完整经验与配置示例：vps-remote-workstation.md](./skills/vps-remote-workstation/references/vps-remote-workstation.md)**

Skill 入口：[SKILL.md](./skills/vps-remote-workstation/SKILL.md)。使用时，将整个 `skills/vps-remote-workstation/` 文件夹放入所用 Agent 的 skills 目录，保留其中的 `references/`。

内容包括：

- 系统基础配置与运行环境
- Mihomo TUN 模式及 SSH / RDP 防失联策略路由
- 链式代理与流量分流配置
- XFCE4 + xrdp 轻量远程桌面
- Firefox ESR 独立配置与登录排障经验
- 验收命令、常见故障与回滚步骤

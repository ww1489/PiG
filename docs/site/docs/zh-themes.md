# 主题

PiG 用主题给终端界面配色。通过 `/settings` 或 `theme` 设置选择主题。

> 本页是英文页 `themes.md` 的中文译本，对应 PiG 提交 `c64bebd8`。英文页是规范来源；两边不一致时以英文页为准。

## 选择主题

启动 PiG 并打开设置：

```text
/settings
```

选择当前主题。PiG 会把选择写入对应的设置文件。

也可以直接在 `~/.pig/agent/settings.json` 里写名字：

```json
{
  "theme": "dark"
}
```

## 完整示例：选一个主题

打开设置列表，输入 `theme` 找到那一行，按回车。在 PiG 上核对过，全新启动显示的是 `system`（用终端自身配色生成的主题）：

```text
> theme
→ Theme                             system
  Default thinking level per model  none
  Color theme for the interface
  Type to search · Enter/Space to change · Esc to cancel
```

```text
Theme
Select a theme, or choose automatic to follow terminal appearance.
→ ✓ system     Theme created from your terminal's colors
    automatic  Use separate themes for light and dark terminal appearance
    dark
    light
  Enter to select · Esc to go back
```

勾号表示正在使用的主题。移到 `dark` 按回车。颜色立刻变化，这一行显示 `Theme  dark`，PiG 已经把选择写进 `~/.pig/agent/settings.json`：

```json
{
  "lastChangelogVersion": "1.2.0",
  "theme": "dark"
}
```

只想临时试一个主题、不保存，用 `--use-theme`：`pig --use-theme light`。自己的主题文件放在 `~/.pig/agent/themes/`，会出现在这个列表里。格式沿用 Pi 的格式，完整颜色键见下面链接的 Pi 主题文档。

PiG 在启动和重载时，用 Pi 的主题 schema 检查每个主题文件。较新的 schema 更严格，所以在旧版本能加载的主题文件现在可能被拒绝。主题文件只能包含文档化的顶层字段：`$schema`、`name`、`appearance`、`vars`、`colors` 和 `export`。`colors` 与 `export` 下的每个键都必须是文档化的颜色 token。自己的可复用颜色放到 `vars` 里。把 `$schema` 指向 `https://pi.dev/schemas/theme.schema.json`，编辑器就能校验和补全。

## 自动切换

在主题子菜单里选 **Automatic**，可以为浅色和深色终端外观分别指定主题。也可以直接写一对名字：

```json
{
  "theme": "light/dark"
}
```

第一个名字是浅色主题，第二个是深色主题。实时切换需要终端支持配色方案通知（DEC mode 2031）。终端外观变化时，PiG 保留这一对名字。选择单个主题（包括通过扩展选择）会关闭自动切换。

## 主题来源

PiG 可以从这些地方加载主题资源：

- `~/.pig/agent/themes/`；
- 受信任项目的资源；
- 配置好的 Package 成员；
- 设置里的显式路径；
- 活动 Piglet 允许的资源集合。

发现项目主题需要项目信任。

## Package 与 Piglet

Package 可以分发主题资源。安装后，主题进入常规资源发现和过滤。

Piglet 可以为某个 Agent 应用选择资源，并控制环境里的自动发现。Piglet 不会复制或改写主题。

Package 永远不会激活 Piglet。

## 重载

在交互式会话里，PiG 会监视所选自定义主题在 `~/.pig/agent/themes/` 下的文件。文件通知会去抖 100 毫秒再重载。文件缺失或无效时，保留上一个有效主题。操作系统监视器出错会停止通知，但不会终止会话；重新选择该主题即可重启监视。

主题预览不会替换活动的监视注册。其他资源目录里的主题、资源配的改动，仍然需要 `/reload`。

修改主题文件或资源配置后执行：

```text
/reload
```

资源重载失败时，保留上一组可用的扩展。主题解析错误会被报告，不会静默换用另一个作者主题。

## 扩展访问

扩展可以通过公开的 UI 上下文列出主题、查看指定主题、请求切换主题。`ctx.ui.theme.sourcePath` 报告活动主题的源文件，即使该主题来自活动注册表没有持有的文件也一样。

子进程扩展收到的是可序列化的主题数据，无法跨进程边界传递活动的宿主主题对象。

## 安全

主题本身是数据，但分发它的 Package 可能同时包含可执行扩展或指令资源。安装前请检查 Package 的完整成员。

## 上游兼容性

在 Go TUI 暴露相同可观察契约的地方，PiG 沿用 Pi 1.4.0 的主题行为。TypeScript 包 `@earendil-works/pi-tui` 是 Pi 的包，不是 PiG 的 Go API。

完整的主题格式与颜色 token 见 [Pi 的主题文档](https://github.com/earendil-works/pi/blob/v1.2.0/packages/coding-agent/docs/themes.md)。

## 相关文档

- [设置](/docs/latest/settings)
- [Packages](zh-packages.md)
- [Piglets](/docs/latest/piglets)
- [终端界面](/docs/latest/tui)

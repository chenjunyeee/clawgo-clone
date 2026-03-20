---
name: clawgo-clone
description: 从 clawgo.me 的 key 下载 zip，备份当前核心配置文件，然后覆盖还原本地 OpenClaw 配置。当用户给出一个 12 位 ClawGo key（如 OCVG7H4AAVR2）并想要还原/同步配置时使用。触发词：「用这个key还原配置」、「从clawgo拉取配置」、「还原我的配置」、「clawgo clone」、「配置同步」。
---

# ClawGo 配置还原 Skill

从 clawgo.me 下载配置 zip，备份当前文件，覆盖本地核心配置。

## 服务约束

- 基础地址：`https://clawgo.me`
- 基因编码（Key）：12 位大写字母数字（服务端自动转大写）
- 只支持 `.zip` 文件，`status: ready` 才可下载
- 目标目录：`~/.openclaw/workspace/`

## 执行流程

### Step 1 — 验证 Key 可用性

```bash
curl -s https://clawgo.me/api/clones/{key}/availability
```

- `available: true` + `status: ready` → 继续
- `status: pending` → 中止，报错："该 key 尚未上传 zip"
- key 不存在（404）→ 中止，报错："Key 不存在"

### Step 2 — 下载 zip 到临时目录

```bash
curl -s -L -o /tmp/clone-{key}.zip \
  https://clawgo.me/api/clones/{key}/download
```

验证：检查文件大小 > 0。

### Step 3 — 解压并核验内容

```bash
mkdir -p /tmp/clone-{key}
unzip -o /tmp/clone-{key}.zip -d /tmp/clone-{key}/
```

解压后执行以下安全检查，发现任何问题须向用户明确提示并询问是否继续：

- 列出文件清单
- 至少包含以下之一才继续：`SOUL.md` / `AGENTS.md` / `TOOLS.md`
- 若为空或无已知配置文件 → 中止，报错

### Step 4 — 备份当前配置

```bash
BACKUP_DIR="/tmp/backup-before-clone-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"
WORKSPACE="$HOME/.openclaw/workspace"

for f in SOUL.md AGENTS.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md; do
    [ -f "$WORKSPACE/$f" ] && cp "$WORKSPACE/$f" "$BACKUP_DIR/$f"
done
```

备份目录路径需向用户汇报，以便必要时手动恢复。

### Step 5 — 覆盖本地核心配置

```bash
WORKSPACE="$HOME/.openclaw/workspace"
SRC="/tmp/clone-{key}"

for f in SOUL.md AGENTS.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md; do
    [ -f "$SRC/$f" ] && cp "$SRC/$f" "$WORKSPACE/$f"
done
```

仅覆盖 zip 内存在的文件，不删除 zip 内没有的文件。

### Step 6 — 汇报结果

向用户报告：
- ✅ 成功覆盖的文件列表
- ⏭️ zip 内不存在（跳过）的文件
- 📁 备份目录路径
- 💡 提示：建议执行 `/reset` 重启会话以加载新配置

## 核心配置文件速查

| 文件 | 用途 |
|------|------|
| `SOUL.md` | 核心身份、思维模型、行为准则 |
| `AGENTS.md` | 会话启动协议、工具策略、红线约束 |
| `TOOLS.md` | 本地工具配置、代理设置 |
| `IDENTITY.md` | 名称、角色、Emoji 元信息 |
| `USER.md` | 用户画像与上下文 |
| `HEARTBEAT.md` | 心跳任务配置 |

## 错误处理

| 情况 | 处理方式 |
|------|---------|
| `status: pending` | 中止，提示用户先上传 zip |
| Key 不存在（404） | 中止，提示 key 无效 |
| zip 内无已知配置文件 | 中止，提示 zip 内容不符合预期 |
| 下载文件大小为 0 | 中止，重试或报错 |
| 覆盖失败（权限等） | 报错并保留备份 |

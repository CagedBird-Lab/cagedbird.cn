# AGENTS.md instructions for cagedbird.cn

本仓库是 cagedbird.cn 的 Hexo/NexT 博客源码。

## 关键文件与配置层级

NexT 主题的配置有三层，优先级从高到低：

1. **`_config.next.yml`**（本仓库根目录）→ 个性化覆盖，优先级最高
2. **`themes/next/_config.yml`** → 主题默认配置（仓库跟踪的文件）
3. **`_config.yml`**（本仓库根目录）→ Hexo 主配置

⚠️ **遇到过的问题**：想要改菜单项时，必须改 `_config.next.yml`（第 36 行 `menu:` 段），`themes/next/_config.yml` 里的设置会被它覆盖掉。之前因为不知道有这个文件，到 `themes/next/_config.yml` 里找问题绕了大半天。

## 关键路径

| 文件/路径 | 说明 |
|-----------|------|
| `_config.next.yml` | NexT 主题覆盖配置（菜单、侧栏、第三方集成都在这里） |
| `_config.yml` | Hexo 主配置 |
| `themes/next/` | NexT 主题（git submodule，注意 `_config.yml` 可能被覆盖） |
| `source/_posts/` | 文章目录 |
| `source/404.html` | 自定义 404 公益页面（宝贝回家），由 Caddy `handle_errors` 触发 |
| `.github/workflows/deploy.yml` | CI/CD：push main → hexo generate → rsync 到 hk-edge |

## 搜索文件

- **优先 `rga -rl '关键词' source/`**，直接指定目录，不经过 `find | xargs` 管道中转。`find` 的 pipe 在中文路径上会崩，而 ripgrep 自己处理 UTF-8 路径完全正常。
- 不想搜内容只想找文件：`rga --files source/` 或 `ls -R source/`。
- 不要 `curl` 去线上网站查文件在哪——本地源码目录直接 `ls` 或 `rga` 更快。

## 基本规则

- 修改要小，范围要准，只动和本次请求相关的文件。
- 文章和页面以 `source/` 下的源码为准，不直接修改生成产物。
- 不提交 `public/`、缓存、部署目录或构建产物，除非用户明确要求。
- 写文章不需要本地跑 `npm run build`；有 CI 负责构建验证。
- 提交或推送前，用 `git status --short` 确认工作区。

## 写文章

- 新文章放在 `source/_posts/`。
- 本站文章按文件系统树形结构分类；新文章应放进语义明确的目录，而不是堆在根目录。
- 保持现有中文技术博客风格：具体、直接、有排障证据。
- front matter 至少包含 `title`、`date`、`tags`、`categories`。
- 引言后使用 `<!-- more -->`。
- 排障类文章优先写清楚：现象、误判、证据、修复、经验。
- 不暴露密码、token、私有密钥、完整敏感 UUID、内网细节等，除非用户明确要求。

## Git

- 只提交和本次任务相关的文件。
- 提交信息优先使用中文，简短说明本次变更，例如：
  - `文章：记录 Arch 休眠修复`
  - `修复：更新站点配置`
  - `维护：调整博客元信息`
- 用户要求推送时，直接推送当前分支。
- 不改写历史，除非用户明确要求。

---
description: 运行时 skill 的安装与加载，skill 是以 ZIP 分发的 Markdown 文档包，安装后注入 agent 上下文。
---

# roku-plugin-skills

> 本文所说的 skill 指 **Roku 运行时 skill**——可安装、可在对话中调用的 ZIP 档。与开发工具链里那种"给 coding agent 使用的 skill 片段"是两件不同的事，后者不是 Roku 代码的一部分。详见 [Glossary](../overview/glossary.md#skill两类)。

Roku 的 skill 是一个带有 `SKILL.md` 清单文件的目录，以 ZIP 归档分发。安装后，skill 内容（Markdown 文档）可以被注入 agent 的对话上下文，从而扩展 agent 能处理的任务范围。这不是别的框架里的 slash command 或 function call，就是把一批文档内容装进上下文。

这个 crate 管理 skill 的安装、查找和文档加载，核心结构体是 `SkillRegistry`。

## 安装流程

```
install_from_url(url, activated_by)
  ├── SkillSource::parse(url)           // 解析来源类型（GitHub / ZIP URL / 本地路径）
  ├── list_skills()                     // 检查是否已安装（幂等）
  ├── fetcher.fetch(source)             // 下载 ZIP 归档
  ├── extract_archive(bytes, temp_dir)  // 解压到临时目录
  ├── resolve_source_dir(root, subpath) // 定位 SKILL.md 目录
  ├── load_descriptor(source_dir, ...)  // 解析 SKILL.md front matter（YAML）
  ├── copy_skill_files(src, dest)       // 复制到 registry 目录
  └── write_install_record(record)      // 持久化 InstalledSkillRecord JSON
```

安装记录存储在 `registry_root/skills/{skill_name}/`，JSON 格式。`ensure_installed_from_url` 是幂等版本，已安装则返回现有记录。

安装幂等性依赖 `skill_source_matches` 函数判断是否已安装，该函数的 GitHub reference 比较逻辑：[未查明]。

## 来源类型

`SkillSource` 支持三种来源：

- **GitHub**：通过 GitHub API 获取 default branch 和 tarball。具体 API 调用路径和 tarball → ZIP 转换逻辑：[未查明]，需阅读 `registry.rs` 的 `HttpSkillArchiveFetcher::fetch` 完整实现。
- **ZIP 归档 URL**：直接 HTTP GET 下载，按 `max_archive_bytes` 限制大小。
- **本地路径**：`SkillSource::local_path(path)` 构造。

分发格式硬编码为 ZIP，不支持 tar.gz（`source.rs` 测试中有明确的 `rejects_unsupported_url` 用例）。

## 接入 ToolRuntime

`SkillRegistry` 通过 `roku-plugin-tools` 的 `build_resource_catalog*` 系列函数接入 tool 系统：

- `list_skills()` 枚举已安装 skill，每个 skill 生成一条 `CatalogDescriptor` 注入 `ResourceCatalog`
- `load_skill_documents(record)` 在执行时读取 skill 的 Markdown 文档内容，供 `SkillRun` tool 注入上下文
- `SkillRegistry::disabled()` 传入时，`SkillRun` tool 被从 catalog 中排除

## 主要类型

**`SkillDescriptor`**：skill 的元数据，字段 `name`、`description`、`version`、`license`、`entrypoint`（`SKILL.md` 相对路径）。

> [未查明] `openai-agents` 格式的 `metadata.json` 中 `interface.display_name` / `interface.short_description` 字段也被解析，但与 `SkillDescriptor` 字段的优先级规则需进一步核查 `registry.rs` 的完整 `load_descriptor` 实现。

**`InstalledSkillRecord`**：安装记录，包含 descriptor、source、`installed_at_unix_ms`、`install_dir`、`installed_files`。`has_scripts()` 判断 skill 是否包含 `scripts/` 子目录——这是路径约定，不是 manifest 字段。

**`SkillsRuntimeConfig`** 可调参数（均支持环境变量覆盖）：

| 参数 | 默认值 | 上限 |
|------|--------|------|
| `http_timeout_seconds` | 30 | 300 |
| `max_archive_bytes` | 32 MB | 128 MB |
| `max_prompt_document_bytes` | 24 KB | 128 KB |
| `max_prompt_documents` | 24 | 128 |

**`SkillArchiveFetcher` trait**：可替换注入，测试时可 mock。生产实现 `HttpSkillArchiveFetcher` 通过 `reqwest::blocking::Client` 下载。

## 当前限制与取舍

下载使用 `reqwest::blocking`，大归档会阻塞调用线程。`SkillRegistry::disabled()` 在测试中大量使用，skill 能力在未配置时默认关闭，而非部分降级。

---
description: 运行时可安装、可加载的 skill 系统，Skill 是以 ZIP 分发的 Markdown 文档集合，安装后注入 agent 上下文以扩展其能力。
---

# roku-plugin-skills

运行时可安装、可加载的 skill 系统。Skill 是以压缩包（ZIP）分发的 Markdown 文档集合，安装后注入 agent 的上下文以扩展其能力。

---

## 1. Purpose

`roku-plugin-skills` 提供：

1. **SkillRegistry**：skill 的安装、查找、列举入口。支持文件后端（`FileBacked`）和禁用模式（`Disabled`）。
2. **SkillSource**：skill 的来源抽象，支持 GitHub repo/tree/blob URL、ZIP 归档 URL、本地路径。
3. **类型定义**（`types.rs`）：`SkillDescriptor`、`InstalledSkillRecord`、`SkillInstallReport`。
4. **错误类型**（`error.rs`）：`SkillRegistryError`，覆盖从 URL 解析到 ZIP 解压的完整错误路径。

来源：`crates/roku-plugins/skills/src/lib.rs`

---

## 2. Crate 依赖

来源：`crates/roku-plugins/skills/Cargo.toml`

| 依赖 | 用途 |
|------|------|
| `roku-common-types` | `CatalogDescriptor`, `ResourceCost`, `ResourceKind`, `ResourceRisk`，以及日志 emit |
| `reqwest` (blocking + rustls-tls) | 从 GitHub / ZIP URL 下载 skill 归档 |
| `serde` / `serde_json` | JSON 元数据序列化/反序列化 |
| `serde_yaml` | YAML 格式的 skill front matter 解析 |
| `tempfile` | 临时目录（安装时解压用） |
| `thiserror` | 错误类型 |
| `url` (with serde) | URL 解析和序列化（`SkillSource` 字段） |
| `walkdir` | 目录遍历（安装时扫描 skill 文件） |
| `zip` | ZIP 归档解压 |

---

## 3. Public Surface

来源：`crates/roku-plugins/skills/src/lib.rs`

```
pub use error::SkillRegistryError;

pub use registry::{
    DownloadedArchive,
    HttpSkillArchiveFetcher,
    SkillArchiveFetcher,      // trait
    SkillRegistry,
    SkillsRuntimeConfig,
    SkillsRuntimeConfigPatch,
};

pub use source::SkillSource;

pub use types::{InstalledSkillRecord, SkillDescriptor, SkillInstallReport};
```

---

## 4. Module Map

### `error.rs`

`SkillRegistryError` 枚举，覆盖所有可能的错误：

- `RegistryDisabled` — 注册表处于 disabled 状态时调用安装/查找
- `InvalidSourceUrl` / `UnsupportedSourceUrl` — URL 解析失败或不支持的来源
- `DownloadFailed` / `ArchiveTooLarge` / `GitHubMetadataFailed` — 下载阶段
- `ArchiveExtract` / `InvalidSkillRoot` / `SkillManifestMissing` — 解压/校验阶段
- `PromptDocumentTooLarge` — 文档超出可注入上下文的大小限制
- `FrontMatter` / `SkillNotFound` / `RuntimeConfig` — 元数据/配置错误
- `Io`, `Json`, `Yaml`, `HttpClient`, `Zip` — 底层 IO 错误 wrap

来源：`crates/roku-plugins/skills/src/error.rs`

### `lib.rs`

crate 入口，将四个子模块的公开类型 re-export。

来源：`crates/roku-plugins/skills/src/lib.rs`

### `registry.rs`

`SkillRegistry` 的核心实现。

**两种后端**（通过内部 `SkillRegistryBackend` enum 切换）：

| 后端 | 构造方式 | 说明 |
|------|----------|------|
| `Disabled` | `SkillRegistry::disabled()` | 所有安装/查找操作返回 `RegistryDisabled` 错误 |
| `FileBacked { root }` | `SkillRegistry::file_backed(root)` | 以本地目录为持久化后端 |

主要方法：
- `install_from_url(source_url, activated_by)` — 解析 URL → 下载 → 解压 → 写入 registry 目录
- `ensure_installed_from_url(...)` — 幂等版本：已安装则返回现有记录
- `list_skills()` — 枚举 registry 目录下所有已安装 skill
- `find_skill(name)` — 按 skill 名精确查找
- `load_skill_documents(record)` — 将已安装 skill 的 Markdown 文档内容读取为字符串列表（注入 agent context 时用）
- `build_catalog_descriptor(name)` — 生成供 `ResourceCatalog` 使用的 `CatalogDescriptor`
- `is_enabled()` — 判断当前是否为 FileBacked 模式

`SkillArchiveFetcher` trait（可替换注入，测试时可 mock）：

```rust
pub trait SkillArchiveFetcher: Send + Sync {
    fn fetch(&self, source: &SkillSource) -> Result<DownloadedArchive, SkillRegistryError>;
}
```

`HttpSkillArchiveFetcher`：生产实现，通过 `reqwest::blocking::Client` 下载。支持 GitHub API（获取默认分支、tarball 等）和直接 ZIP URL。

`SkillsRuntimeConfig` 可调参数：

| 参数 | 默认值 | 上限 | 环境变量覆盖 |
|------|--------|------|-------------|
| `http_timeout_seconds` | 30 | 300 | `ROKU_RUNTIME__SKILLS__HTTP_TIMEOUT_SECONDS` |
| `max_archive_bytes` | 32 MB | 128 MB | `ROKU_RUNTIME__SKILLS__MAX_ARCHIVE_BYTES` |
| `max_prompt_document_bytes` | 24 KB | 128 KB | `ROKU_RUNTIME__SKILLS__MAX_PROMPT_DOCUMENT_BYTES` |
| `max_prompt_documents` | 24 | 128 | `ROKU_RUNTIME__SKILLS__MAX_PROMPT_DOCUMENTS` |

来源：`crates/roku-plugins/skills/src/registry.rs`

### `source.rs`

`SkillSource` 枚举，三个变体：

```rust
pub enum SkillSource {
    LocalPath { path: String, original_url: String },
    GitHub { owner, repo, reference, subpath, original_url },
    ArchiveZip { archive_url, subpath, original_url },
}
```

`SkillSource::parse(url)` 支持的 URL 格式：

| URL 模式 | 解析结果 |
|---------|----------|
| `https://github.com/{owner}/{repo}` | `GitHub`，无 reference/subpath |
| `https://github.com/{owner}/{repo}/tree/{ref}/{path}` | `GitHub`，含 reference 和 subpath |
| `https://github.com/{owner}/{repo}/blob/{ref}/{path}/SKILL.md` | `GitHub`，subpath 为父目录 |
| `https://raw.githubusercontent.com/{owner}/{repo}/{ref}/{path}/SKILL.md` | `GitHub`，subpath 为父目录 |
| `https://example.com/path/archive.zip` | `ArchiveZip` |
| 其他 | `UnsupportedSourceUrl` 错误 |

`version_hint()` 返回 GitHub `reference`（branch/tag/commit）或 `None`。

来源：`crates/roku-plugins/skills/src/source.rs`

### `types.rs`

三个公开类型：

**`SkillDescriptor`** — skill 的元数据：
```rust
pub struct SkillDescriptor {
    pub name: String,
    pub description: String,
    pub version: String,
    pub license: Option<String>,
    pub entrypoint: String,   // SKILL.md 的相对路径
}
```

**`InstalledSkillRecord`** — 安装记录（持久化到 registry 目录）：
```rust
pub struct InstalledSkillRecord {
    pub descriptor: SkillDescriptor,
    pub source: SkillSource,
    pub installed_at_unix_ms: u64,
    pub install_dir: String,
    pub installed_files: Vec<String>,
}
```

`InstalledSkillRecord::has_scripts()` — 判断 skill 是否包含 `scripts/` 目录（用于判断是否支持脚本执行）。

**`SkillInstallReport`** — 安装操作的结果报告（返回给调用方/LLM）：
```rust
pub struct SkillInstallReport {
    pub skill_name: String,
    pub version: String,
    pub source_url: String,
    pub install_dir: String,
    pub installed_files: Vec<String>,
    pub activated_by: String,
    pub message: String,       // 人可读的摘要（含"已安装"/"重复安装"等语义）
}
```

来源：`crates/roku-plugins/skills/src/types.rs`

---

## 5. Skill 类型

在本 crate 中，skill 本质上是一个**带有 `SKILL.md` 清单文件的目录**，以 ZIP 归档分发。`SKILL.md` 包含 YAML front matter（name, description, license）和正文内容（即注入 agent context 的文档内容）。

> [未查明] `openai-agents` 格式的 `metadata.json` 中 `interface.display_name` / `interface.short_description` 字段也被解析（`ParsedOpenAiAgentMetadata`），但与 `SkillDescriptor` 的 name/description 字段的优先级规则需进一步核查 `registry.rs` 的完整 `load_descriptor` 实现。

Skill 安装后存储在 `registry_root/skills/{skill_name}/` 下，安装记录以 JSON 形式持久化。

---

## 6. Registry — 如何注册/发现 Skill

```
安装流程：
install_from_url(url, activated_by)
  ├── SkillSource::parse(url)           // 解析来源类型
  ├── list_skills()                     // 检查是否已安装（幂等）
  ├── fetcher.fetch(source)             // 下载 ZIP 归档
  ├── extract_archive(bytes, temp_dir)  // 解压到临时目录
  ├── resolve_source_dir(root, subpath) // 定位 SKILL.md 目录
  ├── load_descriptor(source_dir, ...)  // 解析 SKILL.md front matter
  ├── copy_skill_files(src, dest)       // 复制到 registry 目录
  └── write_install_record(record)      // 持久化 InstalledSkillRecord JSON
```

Skill 发现通过 `list_skills()` 枚举 `registry_root/skills/` 目录下的安装记录实现。每个已安装 skill 对应一个持久化的 `InstalledSkillRecord` JSON 文件。

`build_catalog_descriptor(skill_name)` 将已安装 skill 的 `InstalledSkillRecord` 转化为 `CatalogDescriptor`（注册进 `ResourceCatalog`），使其对 `roku-plugin-tools` 的 tool builder 可见。

来源：`crates/roku-plugins/skills/src/registry.rs`

---

## 7. Source — Skill 从哪加载

来源：`crates/roku-plugins/skills/src/source.rs`

支持三种来源：

1. **GitHub**（最常见）：
   - 通过 GitHub API 获取 default branch 和 tarball（`resolved_reference` 会被设置为实际 commit SHA 或 tag）
   - > [未查明] 具体 GitHub API 调用路径和 tarball → ZIP 转换逻辑需阅读 `registry.rs` 的 `HttpSkillArchiveFetcher::fetch` 完整实现确认

2. **ZIP 归档 URL**：直接 HTTP GET 下载，按 `max_archive_bytes` 限制大小

3. **本地路径**：`SkillSource::local_path(path)` 构造，由 `fetcher.fetch` 在本地读取

---

## 8. 与 plugins/host / agent-runtime 的装配

来源：`crates/roku-plugins/tools/src/builders.rs`

装配路径：

```
SkillRegistry（来自 roku-plugin-skills）
    │
    ▼（传入 build_resource_catalog*）
ResourceCatalog（包含已安装 skill 的 CatalogDescriptor）
    │
    ├──► ToolRuntime（包含 SkillInstall / SkillRun tool 实例）
    │         │  来自 roku-plugin-tools::builtin::skill_install / skill_execute
    │
    └──► RuntimeVisibleToolAvailabilitySnapshot
              │  snapshot.is_tool_enabled("SkillRun") 等
```

- `roku-plugin-tools` 的 `build_resource_catalog*` 函数接收 `&SkillRegistry` 参数，调用 `skill_registry.list_skills()` 枚举已安装 skill，为每个 skill 生成 `CatalogDescriptor` 注入 `ResourceCatalog`。
- `SkillRun` tool（在 `builtin/skill_execute.rs` 中实现）在执行时通过 `SkillRegistry` 加载 skill 文档，注入 agent 的对话上下文。
- `SkillRegistry::disabled()` 作为"no-op"传入时，`SkillRun` tool 被从 catalog 中排除。

---

## 9. 已知 Tradeoff

- **ZIP 格式唯一**：skill 分发格式硬编码为 ZIP，不支持 tar.gz 等其他归档格式（`source.rs` 测试中有明确的 `rejects_unsupported_url` 用例拒绝 `.tar.gz`）。
- **同步 HTTP**（reqwest blocking）：下载大归档时会阻塞调用线程，对于超过 `HARD_MAX_ARCHIVE_BYTES` (128 MB) 的 skill 包直接报 `ArchiveTooLarge` 错误。
- **Disabled 模式是默认测试状态**：`SkillRegistry::disabled()` 在测试中被大量使用，意味着 skill 能力在未配置时默认关闭，而非部分降级。
- **`has_scripts()` 规则是路径约定**：判断 skill 是否包含脚本仅靠文件路径前缀（`scripts/`），而非 manifest 中的显式字段——这是一个隐式约定，新建 skill 包时需要遵守。
- **安装幂等性依赖 source URL 匹配**：`ensure_installed_from_url` 通过 `skill_source_matches` 函数判断是否已安装，该函数的匹配逻辑（GitHub reference 如何比较等）> [未查明]，需阅读 `registry.rs` 完整实现确认。

---

## 10. Sources

- `crates/roku-plugins/skills/Cargo.toml`
- `crates/roku-plugins/skills/src/lib.rs`
- `crates/roku-plugins/skills/src/error.rs`
- `crates/roku-plugins/skills/src/registry.rs`
- `crates/roku-plugins/skills/src/source.rs`
- `crates/roku-plugins/skills/src/types.rs`
- `crates/roku-plugins/tools/src/availability.rs`（验证 SkillRun enabled 状态的测试）
- `crates/roku-plugins/tools/src/builders.rs`（装配路径）
- `crates/roku-plugins/tools/src/builtin/skill_install.rs`、`skill_execute.rs`（执行侧实现）

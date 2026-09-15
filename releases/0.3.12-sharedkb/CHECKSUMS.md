# Checksums for dsh-raganything-kb 0.3.12-sharedkb (live @ 192.168.8.6:3082)

共享知识库线（sharedkb）的规范发布位置。本版本含**两个**发布件：
插件包 `.tgz`（给 pnpm profile 依赖用）与可移植包 `-portable.tar.gz`（整机一键安装用）。

## 1. dsh-raganything-kb-0.3.12-sharedkb.tgz

插件包（pnpm `file:` 依赖 / `dsh plugin add` 的落地形态）。

```
MD5:    d6c076e84cf89e193afcd50be5502fa1
SHA256: f9c333e46a496fecef822582f5964994eb3a1c1fb2a2f49018feee69ad8b5a8e
SIZE:   5381483 bytes
```

## 2. dsh-raganything-kb-0.3.12-sharedkb-portable.tar.gz

可移植包（安装器 + 载荷 + 插件包 + OFV 资源 + 校验清单）。**新增于 2026-09-15**：
sharedkb 线此前只发 `.tgz`，没有对应 `-portable` 包。

```
MD5:    eed83fc0a2b4b4d49eddd391767a4f40
SHA256: 4ac7aa08d76525c96b6d74fce6e7f57e1f6e0416a606c9cd0f078fdaa5e6d229
SIZE:   10794665 bytes
```

```
tar xzf dsh-raganything-kb-0.3.12-sharedkb-portable.tar.gz
cd dsh-raganything-kb-0.3.12-sharedkb-portable
SUDO_PW='<sudo 密码>' ./install.sh --profile web   # 以 DSH 属主用户运行，不要用 root
./verify.sh --profile web --deep
```

包内结构（216 条目）：

| 路径 | 内容 |
| --- | --- |
| `install.sh` / `uninstall.sh` / `verify.sh` | 幂等安装 / 卸载 / 校验（含 `--deep` 真查询） |
| `manifest.json` | 版本、来源主机、能力清单、打包期修正 |
| `payload/sidecar/server.py` | 0.3.12 补丁版 sidecar（含 `art-` 分支） |
| `payload/sidecar/viewer/` | **OFV 资源 187 项**（pdf.js cmaps / standard_fonts / `ofv.bundle.js`） |
| `payload/{lib,kb}` | 前端 `client.js` / `config.json.template` |
| `plugin/dsh-raganything-kb-0.3.12.tgz` | 插件包（与 §1 同内容） |
| `extras/` | systemd unit、nginx 片段、`sidecar.env.template` |
| `tools/` | `adapt_config.py`、`patch_nginx.py` |

## 构建口径

- **基线**：`0.3.11-sharedkb`（8.6 线上在跑版本），唯一代码差异 = `document_content` 增加
  `art-` 前缀分支（对话产物「打开原始文件」修复）。
- `package/sidecar/server.py` 与 **8.6 线上热修后的活副本字节一致**（2026-09-15 核验）；
  构建器会断言这一点，调用方需传入线上那份 `server.py`。
- 其余全部成员（含 187 个 `sidecar/viewer/` 资源）相对基线**逐字节不变**；
  另移除 0.3.11 包内误带的 `sidecar/server.py.scopefix-20260915-121321.bak`。
- 版本号 `package.json`：`0.3.11` → `0.3.12`；新增 `package/RELEASE-NOTES.md`。
- 构建器：`build/build_rel0312.py`（`build/build_rel0312.py`）、
  `build/build_portable_0312.py`（`build/build_portable_0312.py`）。
  两者均**确定性**打包：固定 mtime/uid/gid、条目排序、gzip `mtime=0`；两次构建 md5 一致，已复核。

## 打包/安装器修复（随本版一起发布）

1. **`install.sh` 补 `payload/sidecar/viewer/` 落地**。此前可移植包安装器只拷
   `sidecar/server.py` 等文件，**漏了 `viewer/`**，导致装出来的实例
   `/viewer/*` 404 —— Office / PDF 产物的「打开原始文件」没有渲染资源可用。
   现为「存在才拷」，对旧载荷保持兼容。
2. **`install.sh` / `uninstall.sh` 的 `pkill` 收窄**（`--no-systemd` 分支）。
   原来是无锚点的 `pkill -f "python.*server.py"`，在多用户宿主上会**连带杀掉**
   systemd 管理的宿主 sidecar **以及**其他用户的懒加载实例。
   2026-09-15 沙盒演练实测触发：宿主 sidecar 靠 `Restart=always` 自愈，
   admin 实例（32001）**没有自愈**，网关直接 502 `rag sidecar on port 32001 unreachable`。
   现锚定到本实例自己的 venv 路径（`$VENV_DIR.*server\.py` / `$RAG_HOME/venv.*server\.py`），
   并把 pid 文件从共享的 `/tmp/raganything-sidecar.pid` 改为 `$RAG_HOME/sidecar.pid`。
   修复后重跑演练：生产 sidecar 列表前后**零差异**。

## 同一资产的其他位置

- `dist/dsh-raganything-kb-0.3.12-sharedkb.tgz`
- `dist/dsh-raganything-kb-0.3.12-sharedkb-portable.tar.gz`

（内容相同，git 去重。）

## 部署状态（2026-09-15 已执行）

8.6 上 **12 份 profile 的依赖已切换**到 0.3.12：

```
"dsh-raganything-kb": "file:/home/lgsj/dsh-raganything-kb-0.3.12-sharedkb.tgz"
```

覆盖宿主 `profiles/web` + 11 个用户（xiongsirui / admin / mason / chenning / …）。
做法是**外科手术式依赖同步**（改 profile `package.json` 依赖 + 替换已安装包的
`package.json` + 补 `RELEASE-NOTES.md`，各留 `.bak-*`），**没有跑 `pnpm install`** ——
8.6 无公网出口，且 `pnpm install` 会清掉现场 `.bak-*` 备份。
执行器：`build/deploy_profiles_0312.py`（默认 dry-run，`--apply` 才落盘，幂等）。

⇒ 「重装会回退到 0.3.11」的风险**已消除**；线上同时跑着热修改的活副本，代码一致。

仍待办：各懒加载用户实例首次启动后各做一次 `/index/rebuild`。

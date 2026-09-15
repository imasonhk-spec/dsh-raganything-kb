# Checksums for dsh-raganything-kb 0.3.13-sharedkb (live @ 192.168.8.6:3082)

共享知识库线（sharedkb）的规范发布位置。本版本含**两个**发布件：
插件包 `.tgz`（给 pnpm profile 依赖用）与可移植包 `-portable.tar.gz`（整机一键安装用）。

## 1. dsh-raganything-kb-0.3.13-sharedkb.tgz

插件包（pnpm `file:` 依赖 / `dsh plugin add` 的落地形态）。

```
MD5:    37fec124b41996b5b5c4ca522f2ae7a6
SHA256: d60b7755be9db0839d74e4159d1487c8b2243e530e53841d0f5ba54971e0ff71
SIZE:   5384086 bytes
```

## 2. dsh-raganything-kb-0.3.13-sharedkb-portable.tar.gz

可移植包（安装器 + 载荷 + 插件包 + OFV 资源 + 校验清单）。

```
MD5:    bc1178477c0fac5405277d7bdfda1e43
SHA256: 69c8acefe15ed31955a64c39db4b79c2da05c1ef7f1cbc76a9ac8c6e1100aac8
SIZE:   10800099 bytes
```

```
tar xzf dsh-raganything-kb-0.3.13-sharedkb-portable.tar.gz
cd dsh-raganything-kb-0.3.13-sharedkb-portable
SUDO_PW='<sudo 密码>' ./install.sh --profile web   # 以 DSH 属主用户运行，不要用 root
./verify.sh --profile web --deep
```

包内结构（205 条目）：

| 路径 | 内容 |
| --- | --- |
| `install.sh` / `uninstall.sh` / `verify.sh` | 幂等安装 / 卸载 / 校验（含 `--deep` 真查询） |
| `manifest.json` | 版本、来源主机、能力清单、打包期修正 |
| `payload/sidecar/server.py` | 0.3.13 补丁版 sidecar（@ 共享作用域修复 + host 守卫） |
| `payload/sidecar/viewer/` | **OFV 资源 187 项**（pdf.js cmaps / standard_fonts / `ofv.bundle.js`） |
| `payload/{lib,kb}` | 前端 `client.js` / `config.json.template` |
| `plugin/dsh-raganything-kb-0.3.13.tgz` | 插件包（与 §1 同内容） |
| `extras/` | systemd unit、nginx 片段、`sidecar.env.template` |
| `tools/` | `adapt_config.py`、`patch_nginx.py` |

## 本版修复：@ 共享文件 / 文件夹 限定范围问答

两层根因，均已线上验证（8.6 admin 实例 @ 共享 PPTX 由 hits=0 → hits=18）：

1. **前缀不对称**：共享 chunk 的 `file_path` 带 `共享/` 前缀，前端 `@` scope 不带；
   旧 `_in_scope` 只剥 `全部文档/`。修复：`_VROOT_PREFIXES` 加 `"共享/"`，
   并对 chunk 与 scope 两端对称剥虚拟根。
2. **作用域检索跑在用户实例本地（系统性主因）**：用户实例 `rag` 只持私有库
   （0 个 `共享/` chunk），共享库本体在宿主 sidecar（17321）。修复：`_route_scope_query`
   **host-aware**——宿主（`_shared_is_local()` 为 True）本地检索；用户实例把命中共享文件夹
   的作用域转发宿主检索、私有部分本地检索并合并（带【共享知识库】/【私有知识库】标签）。
   单文件同时正确服务宿主与用户，不会转发自环。

0.3.12 的 artcontent 修复（`GET /documents/{art-*}/content` 不再 404）原样保留。

## 构建口径

- **基线**：`0.3.12-sharedkb`（8.6 线上在跑版本），唯一代码差异 = `server.py` 的上述
  `@` 共享作用域修复 + `package.json` 版本号。
- `package/sidecar/server.py` = 8.6 线上热修补副本（admin 用户实例那份，已实测命中共享库）
  **＋ host 守卫**（`if _shared_is_local(): 走本地`）。宿主/用户实例共用同一文件，行为正确。
- 其余全部成员（含 187 个 `sidecar/viewer/` 资源）相对基线**逐字节不变**（构建器断言）。
- 版本号 `package.json`：`0.3.12` → `0.3.13`；`package/RELEASE-NOTES.md` 更新为本版说明。
- 构建器：`build/build_rel0313.py`、`build/build_portable_0313.py`，均**确定性**打包
  （固定 mtime/uid/gid、条目排序、gzip `mtime=0`），两次构建 md5 一致，已复核。

## 打包/安装器修复（沿用 0.3.12，本版不变）

1. **`install.sh` 补 `payload/sidecar/viewer/` 落地**（存在才拷，兼容旧载荷）。
2. **`install.sh` / `uninstall.sh` 的 `pkill` 收窄到本实例 venv**，pid 文件改
   `$RAG_HOME/sidecar.pid`；多用户宿主上不再误杀其他用户实例（详见 0.3.12 归档）。

## 同一资产的其他位置

- `dist/dsh-raganything-kb-0.3.13-sharedkb.tgz`
- `dist/dsh-raganything-kb-0.3.13-sharedkb-portable.tar.gz`

（内容相同，git 去重。）

## 部署状态（2026-09-16 发版，待切换）

代码侧：本版 `server.py` 与 8.6 线上热修副本等价（已实测），发布件正确。

部署侧：**12 份 profile 依赖仍指向 0.3.12-sharedkb**（尚未切换到 0.3.13）。
线上当前跑的是热修副本，功能正常；但若有人从 0.3.12 tarball 重装，会回退到「不含转发修复」的
0.3.12（仅前缀修复在基线内）。

⇒ 建议切到 0.3.13 以固化修复：`build/deploy_profiles_0313.py`（外科手术式依赖同步，
默认 dry-run，`--apply` 才落盘，幂等；不改已装 `server.py` 之外的逻辑、不跑 `pnpm install`）。
切换后宿主 `systemctl restart raganything-sidecar`、用户实例 `POST :3200N/models/restart`。

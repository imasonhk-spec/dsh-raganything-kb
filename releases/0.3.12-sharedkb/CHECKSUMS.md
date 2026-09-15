# Checksums for dsh-raganything-kb 0.3.12-sharedkb (live @ 192.168.8.6:3082)

## dsh-raganything-kb-0.3.12-sharedkb.tgz

MD5:    d6c076e84cf89e193afcd50be5502fa1
SHA256: f9c333e46a496fecef822582f5964994eb3a1c1fb2a2f49018feee69ad8b5a8e
SIZE:   5381483 bytes

## 构建口径

- **基线**：`0.3.11-sharedkb`（8.6 线上在跑版本），唯一代码差异 = `document_content` 增加
  `art-` 前缀分支（对话产物「打开原始文件」修复）。
- `package/sidecar/server.py` 与 **8.6 线上热修后的活副本字节一致**（2026-09-15 核验）；
  构建器会断言这一点，调用方需传入线上那份 `server.py`。
- 其余全部成员（含 187 个 `sidecar/viewer/` 资源）相对基线**逐字节不变**；
  另移除 0.3.11 包内误带的 `sidecar/server.py.scopefix-20260915-121321.bak`。
- 版本号 `package.json`：`0.3.11` → `0.3.12`；新增 `package/RELEASE-NOTES.md`。
- 构建器：`build/build_rel0312.py`（确定性打包：固定 mtime/uid/gid、条目排序、
  gzip `mtime=0`；两次构建 md5 一致，已复核）。

## 同一资产的其他位置

同一 blob 亦提交在 `dist/dsh-raganything-kb-0.3.12-sharedkb.tgz`（内容相同，git 去重）。
本目录为 sharedkb 线的规范发布位置（对照 `releases/0.3.10-sharedkb/`）。

## 部署注意

8.6 上 12 份 profile 的依赖当前仍写死
`file:/home/lgsj/dsh-raganything-kb-0.3.11-sharedkb.tgz`。
换用本包需：把 tarball 放到 8.6 → 改这 12 份 `package.json` 的依赖 →
重启 dsh-web 与各用户实例 sidecar。**本版发布时未执行该步骤**（线上跑的是热修副本，
代码已新，只是"重装会回退到 0.3.11"的风险要等改完依赖才消除）。

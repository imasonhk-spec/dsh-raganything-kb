# Release 0.3.12-sharedkb — 对话产物「打开原始文件」修复

> **分支定位**：本包是 **192.168.8.6 多用户网关线（sharedkb）** 的发布件，
> 基线 = 0.3.11-sharedkb。**不是** `src/`（0.4.0-sharedkb.1）的超集，请勿互相合并。
> `src/` 线的产物走「真实入库」模型（`file_path` 带 `DSH产物/`、`知识库产物/` 前缀），
> 不存在本问题，因此本补丁**不适用于 `src/` 线**。

## 现象

知识库面板里点「对话产物 → 打开原始文件」：

```
无法读取原始文件内容：RAG-Anything 无响应或该文档无内容
```

文本型产物（`.md`）能正常打开，**Word / Excel / PPT 类 office 产物必报错**。

## 根因

产物与普通文档的存储模型不同：

| 类型 | doc_id | 落库 |
| --- | --- | --- |
| 上传资料 | LightRAG 真实 doc id | 进 `full_docs` |
| **对话产物** | `art-<uuid>-<文件名>` | **只落盘、不入库** |

sidecar 里两个同族端点对 `art-` 前缀的处理**不一致**：

| 端点 | 0.3.11 | 0.3.12 |
| --- | --- | --- |
| `GET /documents/{doc_id}/file` | ✅ 已处理 `art-` | ✅ |
| `GET /documents/{doc_id}/content` | ❌ 只查 LightRAG `full_docs` → 对 `art-` **恒 404** | ✅ |

前端 `viewDoc`（`lib/client.js`）在产物非文本型时走 `/documents/{id}/content`，
`ragDocContent` 拿到 **null** 就弹出上面那句错误（只有返回 null 才弹）。

## 修复（`sidecar/server.py`，+15 行）

`document_content` 开头增加 `art-` 分支：命中则直接从磁盘读原文 ——
文本型返回正文；二进制型（`docx/xlsx/pptx/pdf/zip/png/jpg/gif/doc/xls/ppt`）返回
空 `content` + `file_path`，交给前端 `ofvUrl` 用 Open File Viewer 渲染（与 `/file` 同源）。

```python
@app.get("/documents/{doc_id}/content")
async def document_content(doc_id: str):
    """Return the original document content stored in full_docs."""
    # 2026-09-15: 仅文件产物（art- 前缀）不进 LightRAG，直接从磁盘读原文；
    # 否则 /documents/{art-}/content 恒 404，前端据此报“无法读取原始文件内容”。
    art = _artifact_by_doc_id(doc_id)
    if art is not None:
        _, _, _phys = art
        try:
            _text = _phys.read_bytes().decode("utf-8")
        except UnicodeDecodeError:
            _text = ""
        if _text == "" or _phys.suffix.lower() in (...):
            return {"doc_id": doc_id, "content": "", "file_path": str(_phys)}
        return {"doc_id": doc_id, "content": _text, "file_path": str(_phys)}
    rag = await ensure_rag()
    ...
```

## 验证记录（2026-09-15）

**修复前（复现）**

- 宿主 `.md` 产物：`/content` = 404，`/file` = 200。
- admin 实例 `.docx` 产物 `art-ac6e5489-…-210048.docx`（21:00:48 生成，即报障对象）：
  `/content` = **404**。

**修复后（走前端真实路径 `https://192.168.8.6:3082/rag/*` → 网关 → admin 实例，复用 admin 会话）**

| 请求 | 结果 |
| --- | --- |
| `GET /rag/documents/art-…docx/content` | **200**（office 型：空正文 + `file_path`） |
| `GET /rag/documents/art-…docx/file` | **200**，37318 B（真实 docx） |
| 同实例 `.md` 产物 `/content`（对照） | **200**，带真实正文 |

**生效范围**：12 份 `server.py`（宿主 + 11 用户实例）全部就地打补丁，各留备份
`.bak-artcontent-20260915-211205`；宿主 `systemctl restart raganything-sidecar`（pid 4087851），
admin 实例 `POST :32001/models/restart`（pid 4098529）。

## 产物

| 项 | 值 |
| --- | --- |
| 文件 | `dist/dsh-raganything-kb-0.3.12-sharedkb.tgz` |
| 大小 | 5 381 483 B |
| md5 | `d6c076e84cf89e193afcd50be5502fa1` |
| sha256 | `f9c333e46a496fecef822582f5964994eb3a1c1fb2a2f49018feee69ad8b5a8e` |

打包器 `tools/_build_rel0312.py`，**确定性**（固定 mtime/uid/gid、条目排序、gzip `mtime=0`，
两次构建 md5 一致）。相对 0.3.11 基线的差异仅三处，已由脚本自查断言：

1. `package/sidecar/server.py` 换成线上已验证的补丁版（301 988 → 302 744 B，与线上字节一致）；
2. `package/package.json` 版本 `0.3.11` → `0.3.12`；
3. 删除 0.3.11 包内误带的 `sidecar/server.py.scopefix-20260915-121321.bak`；
4. 新增 `package/RELEASE-NOTES.md`。

其余 187 个 `sidecar/viewer/` 资源与全部载荷文件**逐字节与 0.3.11 相同**（脚本内 md5 断言）。

## 部署注意（尚未执行）

8.6 上 12 份 profile 的依赖当前仍写死旧包路径：

```
"dsh-raganything-kb": "file:/home/lgsj/dsh-raganything-kb-0.3.11-sharedkb.tgz"
```

换用本包需要：把新包放到 8.6（如 `/home/lgsj/dsh-raganything-kb-0.3.12-sharedkb.tgz`）
→ 12 份 profile 的 `package.json` 改依赖 → 重启 dsh-web 与各用户实例 sidecar。
**本版未执行这一步**，故线上仍是热修的活副本（已在跑新代码），只是"重装会回退到 0.3.11"的风险
要等改完依赖才真正消除。

# build/ — 打包与移植工具

这些脚本把「正在运行的插件」变成「可分发的包」，以及把补丁安全地送到**已经分叉**的部署上。

## 打一个可移植包

| 脚本 | 作用 |
| --- | --- |
| `build_pkg039.py` | **当前版本的打包器**。取上一版的包装骨架（安装器/验证器/卸载器/extras/tools），刷新载荷为 live 插件目录的快照，套用移植补丁，重建 `manifest.json`，打 `plugin/*.tgz` 与便携包，并跑一遍静态校验 + 沙箱安装演练 |
| `build_pkg037.py` / `build_pkg036.py` | 上一版打包器，保留作对照（`build_pkg036` 按 stdout 而非退出码判断 `bash -n`，因此漏掉了 `uninstall.sh` 的 CRLF 缺陷） |
| `run_build039.py` | 把构建器与补丁器投到目标主机执行，并把产物拉回本地 `dist/` |

`build_pkg039.py` 的 9 个步骤：骨架拷贝 → 刷新载荷 → 移植崩溃防护 → 状态栏计数修复 →
索引弹窗计数修复 → 载荷标记断言 →
版本号 → README → 打 tgz → 重建 manifest → 打便携包 → 静态校验 → **沙箱安装演练**。

> **沙箱演练原理**：`install.sh` 用 `RUN_HOME="${HOME}"`（只有 root 才走 `getent`），
> 所以把 `HOME` 指向 `/tmp/...` 就能零风险跑完整安装 + `verify.sh`，完全不碰生产。
> 演练里额外传 `--port 17999`，避免 verify 打到生产 sidecar。

## 打一个发行 tgz（同线增量回灌）

| 脚本 | 作用 |
| --- | --- |
| `build_rel0312.py` | 以**上一版发行 tgz** 为基线，把线上已验证的补丁回灌成新版本包（默认 0.3.11-sharedkb → 0.3.12-sharedkb）：替换 `server.py`、bump `package.json`、清掉包内误带的 `.bak`、补 `RELEASE-NOTES.md`，并**断言其余成员相对基线逐字节不变**。确定性打包（固定 mtime/uid/gid、条目排序、gzip `mtime=0`），同输入同 md5 |

```bash
python build/build_rel0312.py \
    --base   dsh-raganything-kb-0.3.11-sharedkb.tgz \
    --server-py /path/to/patched/sidecar/server.py \
    --out    dist/dsh-raganything-kb-0.3.12-sharedkb.tgz
```

这样打出来的包与线上**热修后的活副本**字节一致，避免"热修在跑、重装回退"的两套状态。

## 把补丁送到已分叉的部署

| 脚本 | 作用 |
| --- | --- |
| `apply_norm_fix.py` | 把面板崩溃防护 `normaliseModelsSnap` 函数级移植到缺少它的分支（逐字节同一段代码，锚点唯一性断言） |
| `apply_count_rail_fix.py` | 把「状态栏计数口径 + 对话产物栏 md 过滤」补丁打到带产物栏的分支（锚点计数断言 + 幂等） |
| `apply_extdist_fix.py` | 把「索引弹窗计数口径」补丁打到带产物栏的分支：`extDist` 由遍历 `docs` 改为 `kbDocs`，依赖数组 `[docs]`→`[kbDocs]`（锚点计数断言 + 幂等） |
| `kb_artifact_rail_patch.py` | 「对话产物」栏 + 产物文件夹下线的完整补丁集（前端布局、产物栏组件、sidecar 检索排除） |
| `deploy_artrail.py` | 把上面这份补丁分别部署到三套部署（含各账户副本），带时间戳备份并重启 DSH |

`deploy_artrail.py` 连非 Windows 主机时**不内置任何口令**，需要先导出环境变量：

```bash
export DSH_SSH_PW_100='<100.100.6.55 的 sudo 口令>'
export DSH_SSH_PW_86='<192.168.8.6 的 sudo 口令>'
python deploy_artrail.py --host all
```

未导出对应变量时脚本会立刻以 `[FAIL] ... is not set` 退出，不会带着空口令去连。

**打补丁的硬规则**：先 `diff` 摸清真实差异，再按函数级锚点做替换，每处断言 `count == 1`。
不要整文件覆盖 —— 各分支都有对方没有的特性，覆盖会静默把它们冲掉。

## 外部前置

这些脚本假定你手上已经有一台可 SSH 的 DSH 主机（`ssh_<host>.py` 提供 paramiko 连接），
以及目标主机上存在上一版的包骨架。它们**不含任何凭据**，主机地址与账号请自行在
`ssh_*.py` 里配置。

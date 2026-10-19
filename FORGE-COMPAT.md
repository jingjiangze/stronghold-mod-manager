# Forge 兼容边界

**结论：Forge 兼容已经做到位了**（审计 2026-10-19/20 实测），不需要为「兼容 Forge」再做架构改动。
剩下的是**把兼容产物接进托管仓库**（那是 mod 仓的事，不是管理器的事）。

## 1. 接受的形态

| 输入 | 是否接受 | 证据 |
|---|---|---|
| zip，根有 `pack.json` | ✅ | `forgeImport.js:1089` `if (!files.has('pack.json')) fail(...)` |
| zip，一层包裹目录内有 `pack.json` | ✅ | `forgeImport.js:544` `stripWrapper()` |
| 目录，含 `pack.json` | ✅ | `forgeImport.js:3200` `else if (st.isDirectory()) files = readUntrustedDir(abs);` |
| zip，是插件包（`plugin.json`） | ✅（走另一条） | `install.js:535` |
| 两者皆无 | ❌ 拒绝 | `import.js:41` |

## 2. 产出形态

`importForge()` 把每个 product 落成一个**标准 manifest-2 插件目录** `plugins/<id>/`：
写 `PLUGIN_MANIFEST`（即 `plugin.json`）→ 数据与素材 → 顶层 `IMPORT-REPORT.md`；
再由 `commitProducts` 把 `stage/<id>` 改名为 `plugins/<id>`。

**与手写插件完全同一形态**——这是关键：Forge 包导入后不需要任何特殊对待，
后续装载、启用、列表展示走的是同一条 `contentSet → room.setPlugins` 链。

证据：
- `server/plugins/forgeImport.js:3162` — `fs.writeFileSync(path.join(dir, PLUGIN_MANIFEST), ...)`
- `server/plugins/forgeImport.js:361` — `fs.renameSync(path.join(stage, p.id), dest);`
- `server/plugins/forgeImport.js:355` — `const dest = path.join(parent, p.id);`（parent = pluginsDir）

## 3. 「兼容的 forge mod 加入 mod 仓库」——已具备 vs 仍缺

### 已可直接复用
- `tools/mod.mjs pack`：把作者目录打成**确定性 zip**，字节稳定，可 pin SHA-256。
- `importForge` 的 `dryRun`：只出报告不落盘，可作 CI 预检（`forgeImport.js:3231`）。
- 双语报告 markdown：既是 `check` 输出，也是 `/mod/upload` 的 JSON 字段。
- 转换产物与手写插件同形态，落 `plugins/<id>/` 即用。

### 仍缺（属于托管仓库要补的）
- **没有「mod 仓库」概念**：现有代码只负责「导入到本机 `plugins/`」，没有登记、索引、commit 步骤。
- 无**版本语义**：只有 `force` 覆盖，没有语义化版本与取代关系。
- 无**字段级转换正确性的对照测试**——现有 `test/mods/import.test.js` 只断言「落成了含 `plugin.json` 的目录」，
  未断言 operators / chess / skills / voices 的映射是否完整。这是「加入仓库前能否相信它玩法全在」的缺口。

## 4. 测试覆盖缺口（重要，决定 CI 能拦什么）

`test/mods/import.test.js` 现有 6 个用例：
sniff 分类与拒绝、Forge 包真实转换 + dryRun 不写盘、inbox 成功移到 `_done`、
inbox 拒绝移到 `_failed` 带 reason、CLI pack 确定性 zip、`/mod` 只认自己的路径且页面自包含。

**未覆盖**：
- `/mod/upload`、`/mod/fetch` 的 HTTP 层（网关/钥匙/同源/body 大小限制）
- `installPluginBytes` 的拒绝码（SIZE / EXISTS / CHECK）
- `fetchHttps` 的地址、协议、重定向规则
- `importForge` 的字段级转换正确性
- ZIP 炸弹 / 符号链接 / 路径穿越等安全校验
- `keyOk` / `changeAllowed` / `studioAllowed` 的门失效路径

⇒ 若要「CI 测试通过才进 mod 文件夹」，这些缺口就是闸门的漏洞面；
建议至少把「拒绝码」与「门失效」两类补上，否则 CI 绿不代表 mod 可信。

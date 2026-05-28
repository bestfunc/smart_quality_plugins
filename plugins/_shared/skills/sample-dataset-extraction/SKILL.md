---
name: sample-dataset-extraction
description: 用户想从某个数据集抽样本看标注（如"抽 NG 样本前 20 条"、"看下数据集 X 里 label=瑕疵 的最近样本"）时使用。批量下载分两条路:有标注走 datasets_export_manifest,无标注原始音频走 datasets_export_files_manifest。
allowed-tools: datasets_list, datasets_get, datasets_query_samples, datasets_get_annotations, datasets_export_manifest, datasets_export_files_manifest, files_presign_download
---

# 抽取数据集样本

## ⚠️ workstation_id 必填

datasets_* 全部 6 个工具(list/get/query_samples/get_annotations/create_draft/import_items)都**必须**传 `workstation_id` —— 数据集、版本绑定、样本通道全在工位库,主库返空。

如果用户没给工位:
- 当前 IDE 上下文里如果有 `X-Workstation-Id`(常见于 SPA 当前工位),直接复用
- 否则**问一句"哪个工位的数据集?"** 别瞎传

## 三种用法 — 先判断:探查 / 批量带标注 / 批量原始音频

| 场景 | 走哪条 | 调用次数 |
|---|---|---|
| **探查/调试** ≤ 20 条样本,只想看几条标注详情 | 走下面"探查流程" | 3 + N(query + get_annotations) |
| **批量导出(有标注)** > 20 条,要"音频 + 标注 JSON" 配对下到本地训练用 | `datasets_export_manifest`(见"批量流程 A")|**1 次** |
| **批量导出(无标注/0 细标)** 原始 wav 整文件下载,不切片不要 JSON | `datasets_export_files_manifest`(见"批量流程 B")|**1 次** |

判错的代价:
- 批量场景走探查流程 → 几百次 `get_annotations` + `presign_download` 每次过 LLM,token 烧上亿
- 0 标注数据集走 `export_manifest` → segment 驱动,total 永远 0,白调

**判定规则**:
- 看 `datasets_get` 返回的 `totalDetailMarkCount`:0 → 走流程 B;>0 且 ≤ 20 条 → 探查;>0 且要批量 → 流程 A

## 探查流程(≤ 20 条)

1. **定位数据集**:如用户已给数据集名/ID,跳到 2;否则先 `datasets_list(workstation_id, q=...)` + 关键字过滤,让用户确认
2. **看数据集结构**:`datasets_get(workstation_id, id)` 拿到 `versions` 数组 + 元数据 —— **不报告完整 schema 给用户**,只挑用户关心的字段(如版本号 + dataCount + label 候选列表)
3. **抽样本**:`datasets_query_samples(workstation_id, version_id, filters, page_size)`
   - `version_id` 必填(从 step 2 的 `versions[*].id` 拿,**不是** dataset.id)
   - `filters` 是 JSON 字符串(如 `'[{"field":"productUniqueCode","operator":"contains","value":"NG"}]'`),不传则全量
   - `page_size` 默认 50,最大 500
4. **看标注详情**(可选):对挑出的样本,对每个 `channel_id` 调 `datasets_get_annotations(workstation_id, channel_id)`
5. **下载原始文件**(可选):样本 channel 里有 `fileID`(音频 .pcm / 图片)、细标 mark 里也有 `fileID`,调 `files_presign_download(workstation_id, file_id)` 拿 15 分钟内有效的下载 URL,客户端直接 curl/wget,不走后端带宽

## 批量流程 A(有标注 — 走 datasets_export_manifest)

**一次 MCP 调用替代成百上千次逐条 get_annotations + presign。**

1. 定位 `workstation_id` + `version_id`(同探查流程 step 1-2)
2. 调 `datasets_export_manifest`:
   ```
   datasets_export_manifest(
     workstation_id="...",
     version_id="...",
     filters='[{"field":"hasDetailMark","operator":"in","value":"[\\"NG2_P13C\\"]"}]',
     mark_codes=["NG2_P13C"],   # 只导这个 markCode 的 segment(post-filter)
     limit=5000,                # 上限 5000,超则 truncated=true
     presign_ttl_seconds=3600,  # URL 1 小时有效
   )
   ```
3. 拿到响应 `{segments: [...]}` 之后 **不要把每条 segment 单独 print** —— 每条 segment 已经包含:
   - `sourceWavUrl`(presigned,15 分钟~24 小时 TTL,看上面配)
   - 全部标注字段(label / markCode / markIndex / sourceStartTime / sourceEndTime / ...)
   - `filenameStem` / `dateDir`(后端建议命名,直接用)
4. 用 Bash 工具一次性跑切片脚本(同 fileID 共用 URL,客户端去重下载):
   ```bash
   mkdir -p ./downloads
   # 步骤 3 拿到的响应写到 ./downloads/manifest.json
   # 写下面这个脚本到 ./downloads/run_export.py 然后 python ./downloads/run_export.py
   ```
   ```python
   # run_export.py
   import json, os, requests, wave
   m = json.load(open("./downloads/manifest.json", encoding="utf-8"))
   src = {}
   for s in m["segments"]:
       if s["fileID"] in src: continue
       p = f"/tmp/src_{s['fileID']}.wav"
       if not os.path.exists(p):
           with requests.get(s["sourceWavUrl"], stream=True, timeout=300) as r, open(p, "wb") as f:
               for c in r.iter_content(64*1024): f.write(c)
       src[s["fileID"]] = p
   for s in m["segments"]:
       d = f"./downloads/output/{s['label']}/{s['dateDir']}"
       os.makedirs(d, exist_ok=True)
       stem = s["filenameStem"]
       with wave.open(src[s["fileID"]], "rb") as w:
           sr, sw, nc = w.getframerate(), w.getsampwidth(), w.getnchannels()
           w.setpos(round(s["sourceStartTime"]*sr))
           data = w.readframes(round((s["sourceEndTime"]-s["sourceStartTime"])*sr))
       with wave.open(f"{d}/{stem}.wav", "wb") as o:
           o.setnchannels(nc); o.setsampwidth(sw); o.setframerate(sr); o.writeframes(data)
       with open(f"{d}/{stem}.json", "w", encoding="utf-8") as f:
           json.dump(s, f, ensure_ascii=False, indent=2)
   print(f"done: {m['total']} pairs")
   ```
5. 报告用户:总条数 / 唯一源 wav 数 / 输出目录(`./downloads/output/<label>/<dateDir>/`)

### 批量流程 A 注意点

- 响应可达 ~1.5MB(5000 segment),一次进上下文,这是预期的 —— 远好于累积 5000 次 LLM 回合
- `truncated=true` 表示命中超过 limit,提醒用户:"还有更多,要不要按 markCode 分批 / 收窄 filter 再调一次?"
- 同 `fileID` 的 segment **必须共享下载**,客户端脚本去重(上面 `src = {}` 那段)
- 工位库 vs 主库:`datasets_export_manifest` 同样必传 `workstation_id`,会走工位库

## 批量流程 B(无标注原始音频 — 走 datasets_export_files_manifest)

**适用场景**:数据集是「原始 wav 集合」,`totalDetailMarkCount=0`(标注页没右键打过任何细标)。
典型如算法复核/语音原料集 —— 拿来直接灌训练管线,不需要标注 JSON,也不需要切片。

1. 定位 `workstation_id` + `version_id`(同探查 step 1-2)
2. 调 `datasets_export_files_manifest`:
   ```
   datasets_export_files_manifest(
     workstation_id="...",
     version_id="...",
     filters=null,                # 一般不用,要按产品/通道筛同 query_samples
     limit=5000,                  # 上限 20000,单数据集 1000-2000 文件常见
     presign_ttl_seconds=3600,
   )
   ```
3. 响应 `{files: [{fileID, originalWav, sourceWavUrl, dateDir, channelCount, ...}]}` —— **每条 = 一个唯一 wav**,已按 fileID 去重
4. Bash 直接 wget/curl,目录用 `dateDir` 或者自定义:
   ```python
   # 同样把 manifest 落盘后跑
   import json, os, requests
   m = json.load(open("./downloads/manifest.json", encoding="utf-8"))
   for f in m["files"]:
       d = f"./downloads/{f['dateDir']}"
       os.makedirs(d, exist_ok=True)
       p = f"{d}/{f['originalWav']}"
       if os.path.exists(p): continue
       with requests.get(f["sourceWavUrl"], stream=True, timeout=300) as r, open(p, "wb") as o:
           for c in r.iter_content(64*1024): o.write(c)
   print(f"done: {m['total']} files")
   ```

### 批量流程 B 注意点

- **不要**对 0 标注数据集调 `datasets_export_manifest`:它是 segment 驱动,total 永远 0,白调一次
- 文件粒度 = 唯一 `fileID`(后端已去重);如果多 channel 共享同 wav,`channelCount > 1`,首个 channel 元数据保留在响应里
- 同样按 `version_id` 切分 —— 不同数据集调多次,天然分目录(如 s1 → DS-...-0020,s2 → DS-...-0022)

## 输出格式

按下面格式给用户简表:

| 数据编号 | label | markCode | 标注时间 | 标注人 |
|---|---|---|---|---|

如用户要详情,再展开 ROI(细标)。

## 注意事项

- 全部 datasets_* 工具都要 `workstation_id`,漏传 server 直接返 schema validation 错误
- `version_id` 跟 `dataset_id` 是两个东西:`datasets_get` 拿 dataset 详情,返回里的 `versions[*].id` 才是 query_samples 要的 version_id
- `datasets_query_samples` 的 `page_size` 默认 50、最大 500
- **不要循环调** `datasets_get_annotations`:超过 20 条改走 `datasets_export_manifest`(一次 MCP 调用搞定全部 segment + URL)
- 跨组织/跨工位样本你**看不到** —— ws_db 物理隔离 + service 层 SQL WHERE 双重隔离

## 反例

❌ 用户问"看下数据集 X" → 直接 `datasets_query_samples(version_id=...)` 不传 workstation_id → 报缺参 / 返空。
❌ 把 `dataset_id` 当 `version_id` 传给 query_samples → 总是 total:0(子查询 `DetectChannelVersion.versionID` 不匹配)。
✅ 先 `datasets_get(workstation_id, id)` 拿版本数组,告诉用户"V1.0 有 6294 条,要按啥条件筛?",再 query_samples 传对应 version.id。

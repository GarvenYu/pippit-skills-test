# xyq-skill

xyq-skill 是一组面向 AI Agent 的技能包（Skills），通过调用[小云雀](https://xyq.jianying.com/) 的创作Agent能力，实现 AI 图片/视频的生成、编辑、风格转换等创作能力。

小云雀是字节跳动/剪映旗下的 AI 综合创作平台，同时服务于人类创作者和 AI Agent。本技能让 Claude Code 等 AI Agent 能够作为"传话人"，将用户的创作需求转发给小云雀后端 Agent 处理。

## 功能特性

| 功能 | 说明 |
|------|------|
| 创建会话 / 发送消息 | 向小云雀发送自然语言指令，生成图片或视频 |
| 查询会话进展 | 增量拉取会话消息，轮询创作进度和产物结果 |
| 上传文件 | 上传图片/视频到小云雀资产库，获取 asset_id 用于编辑和参考 |
| 下载结果 | 批量下载生成的图片/视频到本地，支持并行下载 |

**小云雀平台能力覆盖：**

- **生成**：文生图、文生视频、图生视频、视频续写
- **编辑**：局部修改、元素替换、镜头调整、风格迁移
- **复杂创作**：一句话生成短剧（剧本→分镜→成片）、复刻视频/TVC/宣传片、音乐 MV 生成、产品展示片制作

## 快速安装

通过 `npx skills` 一键安装技能到你的项目中：

```bash
# 交互式选择要安装的技能
npx skills add  -y -g

# 直接安装指定技能
npx skills add  --skill xyq-skill
```

安装完成后，设置环境变量即可使用：

```bash
export XYQ_ACCESS_KEY="your-access-key"
```

## 使用方式

### 1. 鉴权

所有 API 请求通过 HTTP Header 进行 Bearer Token 鉴权：

```
Authorization: Bearer <XYQ_ACCESS_KEY>
```

### 2. 创建会话 / 发送消息

```bash
# 创建新会话
python3 skills/xyq-skill/scripts/submit_run.py --message "生一个动漫视频"

# 向已有会话发送消息
python3 skills/xyq-skill/scripts/submit_run.py --message "再生成一个故事视频" --thread-id THREAD_ID

# 携带参考文件发送
python3 skills/xyq-skill/scripts/submit_run.py --message "参考这个视频做修改" --asset-ids asset_id1 asset_id2
```

**参数：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `--message` | 是 | 创作指令内容 |
| `--thread-id` | 否 | 已有会话 ID，不传则创建新会话 |
| `--asset-ids` | 否 | 资产 ID 列表，支持多个 |

**返回：**

```json
{
  "thread_id": "90f05e0c-...",
  "run_id": "abc123-..."
}
```

### 3. 查询会话进展

```bash
python3 skills/xyq-skill/scripts/get_thread.py --thread-id THREAD_ID --run-id RUN_ID --after-seq 0
```

**参数：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `--thread-id` | 是 | 会话 ID |
| `--run-id` | 否 | 运行 ID |
| `--after-seq` | 否 | 增量拉取起始序号，默认 0 |

**返回：**

```json
{
  "messages": [
    {"id": "1", "role": "user", "content": "生一个动漫视频"},
    {"id": "2", "role": "assistant", "content": [{"type": "...", "subtype": "...", "data": {...}}]},
    {"id": "3", "role": "assistant", "content": [{"type": "...", "data": {"url": "https://..."}}]}
  ]
}
```

### 4. 上传文件

```bash
# 上传图片
python3 skills/xyq-skill/scripts/upload_file.py /path/to/image.png

# 上传视频
python3 skills/xyq-skill/scripts/upload_file.py /path/to/video.mp4
```

仅支持 `image/*` 和 `video/*` 类型，单文件大小限制 200MB。

**返回：**

```json
{
  "asset_id": "asset_xxx"
}
```

### 5. 下载结果

```bash
python3 skills/xyq-skill/scripts/download_results.py \
  --urls URL1 URL2 URL3 \
  --output-dir ./xyq_output \
  --prefix "storyboard" \
  --workers 5
```

**参数：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `--urls` | 是 | 要下载的 URL 列表 |
| `--output-dir` | 否 | 输出目录，默认 `./xyq_output` |
| `--prefix` | 否 | 文件名前缀，如 `storyboard` → `storyboard_01.png` |
| `--workers` | 否 | 并行下载线程数，默认 5 |

**返回：**

```json
{
  "output_dir": "./xyq_output",
  "downloaded": ["./xyq_output/storyboard_01.png", "..."],
  "total": 3
}
```

## 示例

### 场景 1：文生视频（最常见）

```
1. submit_run.py --message "生成一个赛博朋克风格的城市夜景视频"
   → 拿到 thread_id、run_id

2. 每 10 秒轮询：
   get_thread.py --thread-id THREAD_ID --run-id RUN_ID --after-seq SEQUENCE
   → 进行中：展示创作信息，继续轮询
   → 意图确认：展示问题，等用户回复后用同一 thread_id 重新提交
   → 完成：提取产物 URL，下载并展示结果

3. download_results.py --urls URL1 URL2 --output-dir ./output --prefix "cyberpunk"
```

### 场景 2：编辑已有视频

```
1. upload_file.py /path/to/video.mp4
   → 拿到 asset_id

2. submit_run.py --message "把背景换成星空" --asset-ids asset_id

3. 同场景 1 的轮询和下载流程
```

### 场景 3：多参考图生成

```
1. upload_file.py /path/to/ref1.png → asset_id1
2. upload_file.py /path/to/ref2.png → asset_id2
3. upload_file.py /path/to/ref3.mp4 → asset_id3
   （多文件可并行上传）

4. submit_run.py --message "根据参考图和视频生成科普故事视频" --asset-ids asset_id1 asset_id2 asset_id3

5. 同场景 1 的轮询和下载流程
```

### 场景 4：在已有会话中追加需求

```
1. submit_run.py --message "把刚才的视频加个片头" --thread-id EXISTING_THREAD_ID
   → 新的 run_id

2. 同场景 1 的轮询和下载流程
```

### 轮询策略

- **间隔**：每 10 秒查询一次
- **增量拉取**：首次 `--after-seq 0`，后续根据已获取消息数计算新 seq
- **超时**：连续轮询 48 小时无结果则停止
- **错误重试**：单次失败可重试 1 次，连续 3 次失败则停止

## 项目结构

```
pippit-skills-test/
├── LICENSE                         # MIT License
├── README.md                       # 项目说明
└── skills/
    └── xyq-skill/
        ├── SKILL.md                # 技能说明
        ├── .gitignore
        └── scripts/
            ├── _common.py          # 公共模块：配置、API 请求、响应解析
            ├── submit_run.py       # 创建会话 / 发送消息
            ├── get_thread.py       # 查询会话进展
            ├── upload_file.py      # 上传文件到资产库
            └── download_results.py # 批量下载生成结果
```

## 许可证

[MIT](LICENSE)

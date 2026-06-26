# 02交通聚会配套物料 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 制作8项聚会配套物料 — AI效果图 + 可打印HTML + 微信问卷，统一橙红渐变"重返20岁"视觉风格。

**Architecture:** 可打印物料为独立HTML文件（`printables/`），每个文件自包含CSS，浏览器打开后Ctrl+P即可打印。AI效果图通过通义万相ModelScope API生成。趣味问答在问卷星平台配置。

**Tech Stack:** HTML5 + CSS3 (内联样式，无外部依赖), Bash + curl (API调用), Qwen-Image (AI生图), 问卷星 (微信问卷)

**Source of truth:** `docs/superpowers/specs/2026-06-26-reunion-materials-design.md`

---

## File Structure

```
ClassReunion/
├── printables/                          # 新建：可打印物料HTML
│   ├── invitation.html                  # 邀请函（A5对折）
│   ├── sign-in-sheet.html               # 签到表（A4横向）
│   ├── award-certificates.html          # 颁奖证书×6（A4横向）
│   ├── itinerary-card.html              # 行程单（三折页）
│   └── checklist.html                   # 筹备物资清单（A4）
├── generated_images_qwen/               # 新建：AI生成效果图
│   ├── tee_front.png
│   ├── tee_back.png
│   ├── mug_front.png
│   └── mug_back.png
├── generate_qwen.sh                     # 已创建：AI生图脚本
├── photos/                              # 已有：校徽、校园照片
│   ├── 南京工业大学-logo.svg
│   └── 南京工业大学-logo-2048px.png
└── content.json                         # 已有：内容数据源
```

---

### Task 1: AI生成产品效果图

**Files:**
- Modify: `generate_qwen.sh`（已存在，需验证API调用正确）
- Create: `generated_images_qwen/tee_front.png`
- Create: `generated_images_qwen/tee_back.png`
- Create: `generated_images_qwen/mug_front.png`
- Create: `generated_images_qwen/mug_back.png`

- [ ] **Step 1: 测试ModelScope API连通性**

Run:
```bash
curl -s -X POST "https://api-inference.modelscope.cn/v1/images/generations" \
  -H "Authorization: Bearer YOUR_MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen-Image-2512","prompt":"test: a white mug","n":1,"size":"512x512"}' \
  -w "\nHTTP %{http_code}"
```

Expected: HTTP 200 with image data or task_id. If task_id returned (async mode), go to Step 1b. If image data returned directly, skip to Step 2.

- [ ] **Step 1b (if async): 实现异步轮询**

```bash
# 提交任务
TASK=$(curl -s -X POST "$API_URL" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen-Image-2512","prompt":"...","n":1,"size":"1024x1024"}')
TASK_ID=$(echo "$TASK" | python3 -c "import sys,json; print(json.load(sys.stdin)['task_id'])")

# 轮询直到完成
for i in {1..30}; do
  STATUS=$(curl -s "https://api-inference.modelscope.cn/v1/tasks/$TASK_ID" \
    -H "Authorization: Bearer $API_KEY")
  if echo "$STATUS" | grep -q '"status":"SUCCEEDED"'; then
    echo "$STATUS" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['output']['results'][0]['url'])"
    break
  fi
  sleep 5
done
```

- [ ] **Step 2: 更新 generate_qwen.sh 支持异步/同步两种模式**

Replace `generate_qwen.sh` with updated version that handles both sync and async API responses:

```bash
#!/bin/bash
API_KEY="YOUR_MODELSCOPE_API_KEY"
API_URL="https://api-inference.modelscope.cn/v1/images/generations"
MODEL="Qwen/Qwen-Image-2512"
OUTDIR="generated_images_qwen"
mkdir -p "$OUTDIR"

generate() {
  local name="$1" prompt="$2"
  echo "[$name] 提交中..."
  RESP=$(curl -s -X POST "$API_URL" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"$MODEL\",\"prompt\":$(echo "$prompt" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))'),\"n\":1,\"size\":\"1024x1024\"}")
  
  # Check if async (has task_id) or sync (has data)
  TASK_ID=$(echo "$RESP" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('task_id',''))" 2>/dev/null)
  
  if [ -n "$TASK_ID" ]; then
    echo "  异步任务 $TASK_ID, 等待中..."
    for i in {1..60}; do
      sleep 3
      STATUS=$(curl -s "https://api-inference.modelscope.cn/v1/tasks/$TASK_ID" \
        -H "Authorization: Bearer $API_KEY")
      if echo "$STATUS" | grep -q '"SUCCEEDED"'; then
        URL=$(echo "$STATUS" | python3 -c "import sys,json; print(json.load(sys.stdin)['output']['results'][0]['url'])")
        curl -s -o "$OUTDIR/${name}.png" "$URL"
        echo "  完成: $OUTDIR/${name}.png"
        return
      fi
      echo -n "."
    done
    echo "  超时!"
  else
    # Direct response — save image
    echo "$RESP" | python3 -c "import sys,json,base64; d=json.load(sys.stdin); open('$OUTDIR/${name}.png','wb').write(base64.b64decode(d['data'][0]['b64_json']))"
    echo "  完成: $OUTDIR/${name}.png"
  fi
}

# T恤正面
generate "tee_front" "白色纯棉短袖T恤平铺图，正面朝上，产品摄影，白色背景。左上胸蓝色盾形校徽。胸前中央大号橙红渐变圆形，内白色粗体数字20。圆形下方深灰粗体中文'重返20岁'。再下方橙红小字'02交通 NJ Tech'。底部浅灰'2026.7.18'。年轻清新潮牌设计风格。"

# T恤背面
generate "tee_back" "同一件白色T恤背面平铺图，产品摄影，白色背景。背部中央偏上蓝色盾形校徽。下方深灰中文'那年的少年 今天的我们'，带小星星装饰。细线分隔三行浅灰小字'南京工业大学 / 2006 — 2026 / 2026.7.18'。简洁留白设计。"

# 马克杯正面
generate "mug_front" "白色陶瓷马克杯正面视图，圆柱形C形把手，杯身暖色米白渐变。产品摄影白底柔光。正面中央大号橙红渐变圆形内白字20，下方深灰粗体'重返20岁'，橙红小字'02交通 NJ Tech'，底部浅灰'2026.7.18'。年轻温暖设计。"

# 马克杯背面
generate "mug_back" "同款陶瓷马克杯背面视图，杯身暖色渐变，C形把手可见。背面：上方蓝色盾形校徽，深灰走心文案'那年的少年 今天的我们'带星装饰，细线分隔三行浅灰'南京工业大学 / 2006 — 2026 / 2026.7.18'。产品摄影风格。"

echo "===== 全部完成 ====="
ls -la "$OUTDIR/"
```

- [ ] **Step 3: 运行生图脚本**

Run: `bash generate_qwen.sh`
Expected: 4张PNG保存到 `generated_images_qwen/`

- [ ] **Step 4: 检查生成质量**

打开 `generated_images_qwen/` 查看4张图，确认：
- 中文文字是否清晰可读
- 配色是否接近橙红渐变
- 整体构图是否合理
If quality poor, retry with adjusted prompts or fall back to Pollinations.ai.

- [ ] **Step 5: Commit**

```bash
git add -f generate_qwen.sh generated_images_qwen/
git commit -m "feat: add AI-generated product images for T-shirt and mug

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: 创建邀请函 (printable HTML)

**Files:**
- Create: `printables/invitation.html`

- [ ] **Step 1: 创建可打印邀请函**

Create `printables/invitation.html` — A5对折设计，浏览器Ctrl+P打印。Page 1为封面，Page 2为内页详情。使用 `@page` 和 `@media print` 控制打印输出。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>重返20岁 · 聚会邀请函</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
    background: #f5f5f0;
    display: flex; flex-direction: column; align-items: center;
    padding: 20px; gap: 20px;
  }
  .page {
    width: 148mm; height: 210mm; /* A5 */
    background: white; border-radius: 8px;
    box-shadow: 0 2px 16px rgba(0,0,0,0.08);
    display: flex; flex-direction: column; align-items: center;
    justify-content: center; position: relative;
    overflow: hidden; page-break-after: always;
  }
  .page::before {
    content: ''; position: absolute; top: -60px; right: -60px;
    width: 200px; height: 200px; border-radius: 50%;
    background: radial-gradient(circle, rgba(255,107,107,0.08) 0%, transparent 70%);
  }
  /* Cover page */
  .cover .gradient-circle {
    width: 120px; height: 120px;
    background: linear-gradient(135deg, #ff6b6b, #ffa94d);
    border-radius: 50%; display: flex; align-items: center;
    justify-content: center; margin-bottom: 24px;
    box-shadow: 0 8px 32px rgba(255,107,107,0.3);
  }
  .cover .gradient-circle span {
    font-size: 60px; font-weight: 900; color: white;
  }
  .cover h1 {
    font-size: 28px; color: #333; letter-spacing: 4px;
  }
  .cover h2 {
    font-size: 16px; color: #ff6b6b; font-weight: 600;
    margin-top: 4px; letter-spacing: 2px;
  }
  .cover .divider {
    width: 60px; height: 2px;
    background: linear-gradient(90deg, #ff6b6b, #ffa94d);
    margin: 16px auto;
  }
  .cover .cta {
    margin-top: 20px; font-size: 18px; color: #333;
    font-weight: 700; letter-spacing: 2px;
    background: linear-gradient(135deg, rgba(255,107,107,0.1), rgba(255,169,77,0.1));
    padding: 10px 28px; border-radius: 24px;
  }
  /* Inner page */
  .inner {
    justify-content: flex-start; padding: 40px 32px;
    text-align: left; align-items: stretch;
  }
  .inner h3 {
    font-size: 20px; color: #333; letter-spacing: 3px;
    text-align: center; margin-bottom: 28px;
  }
  .info-row {
    display: flex; padding: 12px 0;
    border-bottom: 1px dashed #eee; align-items: flex-start;
  }
  .info-row:last-child { border: none; }
  .info-label {
    color: #ff6b6b; font-size: 14px; font-weight: 600;
    width: 56px; flex-shrink: 0;
  }
  .info-value { color: #555; font-size: 14px; line-height: 1.6; }
  .schedule-preview {
    margin-top: 20px; padding: 16px;
    background: #fdf6f3; border-radius: 8px;
  }
  .schedule-preview h4 {
    font-size: 14px; color: #ff6b6b; margin-bottom: 10px;
  }
  .schedule-preview p {
    font-size: 12px; color: #666; line-height: 1.8;
  }
  .inner .cta-bottom {
    text-align: center; margin-top: auto; padding-top: 24px;
    font-size: 16px; color: #ff6b6b; font-weight: 700;
  }
  /* Print */
  @media print {
    body { background: white; padding: 0; gap: 0; }
    .page {
      box-shadow: none; border-radius: 0;
      width: 148mm; height: 210mm;
    }
  }
</style>
</head>
<body>

<!-- Cover -->
<div class="page cover">
  <div class="gradient-circle"><span>20</span></div>
  <h1>重返20岁</h1>
  <h2>02级交通工程 · 毕业20周年聚会</h2>
  <div class="divider"></div>
  <p style="color:#888;font-size:14px;">南京工业大学 · 江浦校区</p>
  <div class="cta">7月18日，不见不散。</div>
</div>

<!-- Inner page -->
<div class="page inner">
  <h3>聚 会 邀 请 函</h3>
  <div class="info-row">
    <span class="info-label">时间</span>
    <span class="info-value">2026年7月18日（周六）— 19日（周日）</span>
  </div>
  <div class="info-row">
    <span class="info-label">地点</span>
    <span class="info-value">南京工业大学（江浦校区）</span>
  </div>
  <div class="info-row">
    <span class="info-label">人员</span>
    <span class="info-value">02级交通工程全体同学</span>
  </div>
  <div class="info-row">
    <span class="info-label">预算</span>
    <span class="info-value">人均约 700 元（含住宿、餐饮、纪念品）</span>
  </div>
  <div class="schedule-preview">
    <h4>📋 行程概要 · 两天一夜</h4>
    <p>
      <b>Day 1 (7/18)</b>：报到签到 → 怀旧午餐 → 校园导览 → 茶话会 → 晚宴+颁奖 → 夜聊<br>
      <b>Day 2 (7/19)</b>：早餐 → 拜访老师/自由活动 → 告别午餐 → 返程
    </p>
  </div>
  <div class="cta-bottom">✦ 那年从南工大出发，如今归来仍是少年 ✦</div>
</div>

</body>
</html>
```

- [ ] **Step 2: 浏览器预览并测试打印**

Open `printables/invitation.html` in browser.
Verify: two A5 pages display correctly, gradient circle renders, colors match spec.
Press Ctrl+P → check print preview fits A5 layout.

- [ ] **Step 3: Commit**

```bash
git add -f printables/invitation.html
git commit -m "feat: add printable invitation with cover and inner page

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 3: 创建签到表 (printable HTML)

**Files:**
- Create: `printables/sign-in-sheet.html`

- [ ] **Step 1: 创建趣味签到表**

Create `printables/sign-in-sheet.html` — A4横向，18人签到墙+贴便签区。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>02交通签到墙</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
    background: #f5f5f0; display: flex; justify-content: center;
    padding: 20px;
  }
  .sheet {
    width: 297mm; height: 210mm; /* A4 landscape */
    background: white; padding: 28px 32px;
    position: relative; overflow: hidden;
  }
  .sheet::before {
    content: ''; position: absolute; top: -40px; right: -40px;
    width: 150px; height: 150px; border-radius: 50%;
    background: radial-gradient(circle, rgba(255,107,107,0.06) 0%, transparent 70%);
  }
  .header {
    text-align: center; margin-bottom: 20px; position: relative; z-index: 1;
  }
  .header h1 {
    font-size: 24px; color: #333; letter-spacing: 3px;
  }
  .header .sub {
    font-size: 13px; color: #ff6b6b; font-weight: 600; margin-top: 4px;
  }
  .grid {
    display: grid; grid-template-columns: repeat(6, 1fr);
    gap: 10px; position: relative; z-index: 1;
  }
  .slot {
    border: 1.5px dashed #e0ddd5; border-radius: 8px;
    padding: 10px 8px; min-height: 90px;
    display: flex; flex-direction: column;
    background: #fdfdfb;
  }
  .slot .name-line {
    border-bottom: 1.5px solid #ff6b6b; padding-bottom: 4px;
    margin-bottom: 6px; font-size: 12px; color: #999;
  }
  .slot .note-area {
    flex: 1; font-size: 9px; color: #ccc;
    display: flex; align-items: flex-start;
  }
  .footer {
    text-align: center; margin-top: 16px;
    font-size: 14px; color: #888; position: relative; z-index: 1;
    letter-spacing: 2px;
  }
  @media print {
    body { background: white; padding: 0; }
    .sheet { box-shadow: none; }
  }
</style>
</head>
<body>
<div class="sheet">
  <div class="header">
    <h1>猜猜我是谁？—— 02交通签到墙</h1>
    <p class="sub">20年了，你还认得出我吗？</p>
  </div>
  <div class="grid">
    <!-- 18 slots -->
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
    <div class="slot">
      <div class="name-line">签名：_______________</div>
      <div class="note-area">贴便签 / 留言：</div>
    </div>
  </div>
  <p class="footer">写出你对 ta 的第一印象 | 活动后扫描存档 | 02交通 · 毕业20周年</p>
</div>
</body>
</html>
```

- [ ] **Step 2: 浏览器预览**

Open `printables/sign-in-sheet.html` in browser.
Verify: 6×3 grid fits A4 landscape, 18 slots visible, red accent colors correct.

- [ ] **Step 3: Commit**

```bash
git add -f printables/sign-in-sheet.html
git commit -m "feat: add interactive sign-in sheet with 18 slots

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 4: 创建颁奖证书 (printable HTML)

**Files:**
- Create: `printables/award-certificates.html`

- [ ] **Step 1: 创建6张搞笑奖状**

Create `printables/award-certificates.html` — A4横向，每页1张证书，6页。大字报风格，橙红标题。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>02交通 · 最佳XX颁奖</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
    background: #f5f5f0; display: flex; flex-direction: column;
    align-items: center; padding: 20px; gap: 20px;
  }
  .cert {
    width: 297mm; height: 210mm; /* A4 landscape */
    background: white; display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    position: relative; overflow: hidden;
    page-break-after: always;
  }
  .cert::before {
    content: ''; position: absolute; inset: 20px;
    border: 3px dashed #ffe0d5; border-radius: 8px; pointer-events: none;
  }
  .cert .ribbon {
    width: 100%; height: 6px;
    background: linear-gradient(90deg, #ff6b6b, #ffa94d, #ff6b6b);
    position: absolute; top: 40px; left: 0;
  }
  .cert .number {
    position: absolute; top: 60px; right: 50px;
    font-size: 120px; font-weight: 900; color: rgba(255,107,107,0.06);
  }
  .cert .award-name {
    font-size: 42px; font-weight: 900; color: #e8533f;
    letter-spacing: 6px; margin-bottom: 8px;
    position: relative; z-index: 1;
  }
  .cert .award-sub {
    font-size: 16px; color: #666; margin-bottom: 28px;
    letter-spacing: 2px; position: relative; z-index: 1;
  }
  .cert .recipient {
    font-size: 28px; color: #333; font-weight: 700;
    border-bottom: 2px solid #ff6b6b; padding: 4px 40px 8px;
    margin-bottom: 24px; position: relative; z-index: 1;
    min-width: 160px; text-align: center;
  }
  .cert .punchline {
    font-size: 18px; color: #555; text-align: center;
    max-width: 500px; line-height: 1.8; position: relative; z-index: 1;
  }
  .cert .date-line {
    position: absolute; bottom: 50px; font-size: 14px; color: #999;
    letter-spacing: 2px;
  }
  .cert .logo-mark {
    position: absolute; bottom: 30px; right: 40px;
    font-size: 12px; color: #ddd; letter-spacing: 1px;
  }
  @media print {
    body { background: white; padding: 0; gap: 0; }
    .cert { box-shadow: none; }
  }
</style>
</head>
<body>

<!-- Certificate 1 -->
<div class="cert">
  <div class="ribbon"></div>
  <div class="number">01</div>
  <div class="award-name">「变化最大奖」</div>
  <div class="award-sub">The Biggest Transformation</div>
  <div class="recipient">________________</div>
  <p class="punchline">
    20年前你是___，20年后你___。<br>
    感谢你证明了人是会变的这条真理。<br>
    全班同学表示：震惊且欣慰。
  </p>
  <div class="date-line">2026.7.18 · 02交通毕业20周年</div>
  <div class="logo-mark">重返20岁</div>
</div>

<!-- Certificate 2 -->
<div class="cert">
  <div class="ribbon"></div>
  <div class="number">02</div>
  <div class="award-name">「几乎没变奖」</div>
  <div class="award-sub">The Time Traveler</div>
  <div class="recipient">________________</div>
  <p class="punchline">
    20年过去了，你依然___。<br>
    岁月这把杀猪刀，唯独放过了你。<br>
    全班嫉妒且欣慰。
  </p>
  <div class="date-line">2026.7.18 · 02交通毕业20周年</div>
  <div class="logo-mark">重返20岁</div>
</div>

<!-- Certificate 3 -->
<div class="cert">
  <div class="ribbon"></div>
  <div class="number">03</div>
  <div class="award-name">「最远赶来奖」</div>
  <div class="award-sub">The Longest Journey</div>
  <div class="recipient">________________</div>
  <p class="punchline">
    跨越___公里，只为一聚。<br>
    这份情谊，全班敬你一杯。<br>
    你的到来，让这场重逢完整了。
  </p>
  <div class="date-line">2026.7.18 · 02交通毕业20周年</div>
  <div class="logo-mark">重返20岁</div>
</div>

<!-- Certificate 4 -->
<div class="cert">
  <div class="ribbon"></div>
  <div class="number">04</div>
  <div class="award-name">「最佳存在感奖」</div>
  <div class="award-sub">The Unforgettable Presence</div>
  <div class="recipient">________________</div>
  <p class="punchline">
    大学四年你可能话不多，<br>
    但20年后大家第一个想起的就是你。<br>
    有些存在感，时间越久越清晰。
  </p>
  <div class="date-line">2026.7.18 · 02交通毕业20周年</div>
  <div class="logo-mark">重返20岁</div>
</div>

<!-- Certificate 5 -->
<div class="cert">
  <div class="ribbon"></div>
  <div class="number">05</div>
  <div class="award-name">「最强记性奖」</div>
  <div class="award-sub">The Human Archive</div>
  <div class="recipient">________________</div>
  <p class="punchline">
    20年前的细节你如数家珍，<br>
    人肉班级档案库，非你莫属。<br>
    你是我们共同记忆的守护者。
  </p>
  <div class="date-line">2026.7.18 · 02交通毕业20周年</div>
  <div class="logo-mark">重返20岁</div>
</div>

<!-- Certificate 6 -->
<div class="cert">
  <div class="ribbon"></div>
  <div class="number">06</div>
  <div class="award-name">「终身成就奖」</div>
  <div class="award-sub">The Lifetime Achievement</div>
  <div class="recipient">________________</div>
  <p class="punchline">
    不解释。<br>
    你值得。<br>
    全班起立鼓掌。
  </p>
  <div class="date-line">2026.7.18 · 02交通毕业20周年</div>
  <div class="logo-mark">重返20岁</div>
</div>

</body>
</html>
```

- [ ] **Step 2: 浏览器预览**

Open `printables/award-certificates.html` in browser.
Verify: 6 certificates display, orange-red headers visible, recipient name lines present, each fits A4 landscape.

- [ ] **Step 3: Commit**

```bash
git add -f printables/award-certificates.html
git commit -m "feat: add 6 funny award certificates for banquet

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 5: 创建行程单三折页 (printable HTML)

**Files:**
- Create: `printables/itinerary-card.html`

- [ ] **Step 1: 创建三折页行程单**

Create `printables/itinerary-card.html` — 三折页设计，A4横向分为3栏，折叠后约10×20cm口袋尺寸。正面为外页（封面+背面信息），背面为内页（两天行程）。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>重返20岁 · 行程单</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
    background: #f5f5f0; display: flex; justify-content: center;
    padding: 20px;
  }
  .card {
    width: 297mm; height: 210mm; /* A4 landscape */
    background: white; display: flex; page-break-after: always;
  }
  /* Three panels */
  .panel {
    flex: 1; padding: 24px 18px; position: relative;
    border-right: 1px dashed #eee; display: flex;
    flex-direction: column;
  }
  .panel:last-child { border-right: none; }
  /* Panel 1: Cover */
  .panel-cover {
    background: linear-gradient(180deg, #fff8f5 0%, #fff 100%);
    align-items: center; justify-content: center; text-align: center;
  }
  .panel-cover .circle {
    width: 90px; height: 90px;
    background: linear-gradient(135deg, #ff6b6b, #ffa94d);
    border-radius: 50%; display: flex; align-items: center;
    justify-content: center; margin-bottom: 16px;
    box-shadow: 0 6px 24px rgba(255,107,107,0.25);
  }
  .panel-cover .circle span {
    font-size: 44px; font-weight: 900; color: white;
  }
  .panel-cover h2 {
    font-size: 20px; color: #333; letter-spacing: 3px;
  }
  .panel-cover .subtitle {
    font-size: 12px; color: #ff6b6b; font-weight: 600;
    margin-top: 6px; letter-spacing: 2px;
  }
  .panel-cover .divider {
    width: 40px; height: 2px;
    background: linear-gradient(90deg, #ff6b6b, #ffa94d);
    margin: 14px auto;
  }
  .panel-cover .info-mini {
    font-size: 11px; color: #888; line-height: 1.8;
  }
  /* Panel 2: Day 1 */
  .panel-day { justify-content: flex-start; }
  .panel-day h3 {
    font-size: 16px; color: #ff6b6b; margin-bottom: 14px;
    letter-spacing: 2px; text-align: center;
  }
  .panel-day .event {
    margin-bottom: 12px; padding-bottom: 10px;
    border-bottom: 1px solid #f5f0ed;
  }
  .panel-day .event .time {
    font-size: 10px; color: #ff6b6b; font-weight: 700;
  }
  .panel-day .event .title {
    font-size: 12px; color: #333; font-weight: 600;
    margin: 2px 0;
  }
  .panel-day .event .desc {
    font-size: 10px; color: #999;
  }
  /* Panel 3: Day 2 + Back info */
  .panel-back h3 {
    font-size: 16px; color: #ff6b6b; margin-bottom: 14px;
    letter-spacing: 2px; text-align: center;
  }
  .panel-back .qr-placeholder {
    width: 80px; height: 80px; background: #f8f8f8;
    border: 1px dashed #ddd; border-radius: 8px;
    margin: 16px auto; display: flex; align-items: center;
    justify-content: center; font-size: 10px; color: #ccc;
  }
  .panel-back .web-link {
    text-align: center; font-size: 11px; color: #888; margin-top: 8px;
  }
  .panel-back .emergency {
    margin-top: auto; font-size: 10px; color: #aaa;
    text-align: center; padding-top: 12px;
    border-top: 1px dashed #eee;
  }
  /* Print */
  @media print {
    body { background: white; padding: 0; }
    .card { box-shadow: none; }
  }
</style>
</head>
<body>

<!-- Front side: Cover + Back info -->
<div class="card">
  <!-- Panel 1: Cover -->
  <div class="panel panel-cover">
    <div class="circle"><span>20</span></div>
    <h2>重返20岁</h2>
    <p class="subtitle">两天一夜 · 重返校园时光</p>
    <div class="divider"></div>
    <p class="info-mini">
      02级交通工程<br>
      毕业20周年聚会<br>
      2026.7.18 — 7.19
    </p>
  </div>
  <!-- Panel 2: Day 1 -->
  <div class="panel panel-day">
    <h3>Day 1 · 7/18 (周六)</h3>
    <div class="event">
      <div class="time">10:00-12:00</div>
      <div class="title">🎒 报到签到</div>
      <div class="desc">校门口集合，领装备包</div>
    </div>
    <div class="event">
      <div class="time">12:00-13:30</div>
      <div class="title">🍜 怀旧午餐</div>
      <div class="desc">食堂重温学生套餐</div>
    </div>
    <div class="event">
      <div class="time">14:00-16:00</div>
      <div class="title">🚶 校园导览</div>
      <div class="desc">教学楼→图书馆→宿舍→操场</div>
    </div>
    <div class="event">
      <div class="time">16:00-17:30</div>
      <div class="title">☕ 主题茶话会</div>
      <div class="desc">老照片回顾+分享20年</div>
    </div>
    <div class="event">
      <div class="time">18:00-21:00</div>
      <div class="title">🍽️ 正式晚宴</div>
      <div class="desc">问答+颁奖+大合照</div>
    </div>
    <div class="event">
      <div class="time">21:00-深夜</div>
      <div class="title">🌙 自由夜聊</div>
      <div class="desc">酒店续摊，想聊继续</div>
    </div>
  </div>
  <!-- Panel 3: Day 2 + Info -->
  <div class="panel panel-back">
    <h3>Day 2 · 7/19 (周日)</h3>
    <div class="event">
      <div class="time">09:00-09:30</div>
      <div class="title">🥐 早餐</div>
      <div class="desc">酒店早餐</div>
    </div>
    <div class="event">
      <div class="time">09:30-11:00</div>
      <div class="title">👨‍🏫 拜访老师</div>
      <div class="desc">自愿前往 / 校园漫步</div>
    </div>
    <div class="event">
      <div class="time">11:30-13:00</div>
      <div class="title">🍜 告别午餐</div>
      <div class="desc">简单吃一顿，互道珍重</div>
    </div>
    <div class="event">
      <div class="time">13:00起</div>
      <div class="title">👋 各自返程</div>
      <div class="desc">下次再见！</div>
    </div>
    <div class="qr-placeholder">微信群<br>二维码</div>
    <p class="web-link">🔗 聚会网站见群公告</p>
    <p class="emergency">紧急联系：见微信群公告 | 聚会当天联络人：___</p>
  </div>
</div>

</body>
</html>
```

- [ ] **Step 2: 预览并测试**

Open `printables/itinerary-card.html` in browser.
Verify: Three equal-width columns, gradient circle on left, Day 1 center, Day 2 right.

- [ ] **Step 3: Commit**

```bash
git add -f printables/itinerary-card.html
git commit -m "feat: add tri-fold itinerary card with two-day schedule

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 6: 创建筹备物资清单 (printable HTML)

**Files:**
- Create: `printables/checklist.html`

- [ ] **Step 1: 创建打勾清单**

Create `printables/checklist.html` — A4纵向，四大类物资，每项前有可打勾的方框。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>02交通 · 筹备物资清单</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
    background: #f5f5f0; display: flex; justify-content: center;
    padding: 20px;
  }
  .sheet {
    width: 210mm; min-height: 297mm; /* A4 portrait */
    background: white; padding: 36px 40px;
  }
  .sheet h1 {
    font-size: 22px; color: #333; text-align: center;
    letter-spacing: 3px; margin-bottom: 4px;
  }
  .sheet .sub {
    text-align: center; font-size: 12px; color: #ff6b6b;
    font-weight: 600; margin-bottom: 28px; letter-spacing: 2px;
  }
  .group { margin-bottom: 24px; }
  .group h3 {
    font-size: 15px; color: #333; margin-bottom: 10px;
    padding-bottom: 6px; border-bottom: 2px solid #ff6b6b;
    display: flex; align-items: center; gap: 8px;
  }
  .group h3 .icon { font-size: 18px; }
  .item {
    display: flex; align-items: center; gap: 10px;
    padding: 6px 0; font-size: 13px; color: #555;
  }
  .item .checkbox {
    width: 18px; height: 18px; border: 2px solid #059669;
    border-radius: 3px; flex-shrink: 0;
    display: flex; align-items: center; justify-content: center;
  }
  .note {
    margin-top: 28px; padding: 12px 16px;
    background: #f0fdf4; border-radius: 8px;
    font-size: 12px; color: #059669; line-height: 1.8;
  }
  @media print {
    body { background: white; padding: 0; }
    .sheet { box-shadow: none; }
  }
</style>
</head>
<body>
<div class="sheet">
  <h1>筹备物资清单</h1>
  <p class="sub">逐项打勾 · 万无一失 · 02交通毕业20周年</p>

  <div class="group">
    <h3><span class="icon">🎒</span> 装备包（每人一份 ×18）</h3>
    <div class="item"><span class="checkbox"></span> 定制T恤 ×18（提前收集尺码）</div>
    <div class="item"><span class="checkbox"></span> 名牌/胸牌 ×18</div>
    <div class="item"><span class="checkbox"></span> 徽章/纪念品 ×18</div>
    <div class="item"><span class="checkbox"></span> 行程单（三折页） ×18</div>
  </div>

  <div class="group">
    <h3><span class="icon">☕</span> 茶话会</h3>
    <div class="item"><span class="checkbox"></span> 老照片PPT/视频</div>
    <div class="item"><span class="checkbox"></span> 教室/会议室预订确认</div>
    <div class="item"><span class="checkbox"></span> 投影仪 + 音响（确认场地提供）</div>
    <div class="item"><span class="checkbox"></span> 水果 ×3盘 / 零食 ×5袋</div>
    <div class="item"><span class="checkbox"></span> 瓶装水 ×1箱 / 饮料 ×1箱</div>
    <div class="item"><span class="checkbox"></span> 一次性杯子 ×1包</div>
  </div>

  <div class="group">
    <h3><span class="icon">🍽️</span> 晚宴</h3>
    <div class="item"><span class="checkbox"></span> 餐厅预订确认</div>
    <div class="item"><span class="checkbox"></span> 趣味问答二维码（打印张贴）</div>
    <div class="item"><span class="checkbox"></span> 颁奖证书 ×6（打印+留空姓名）</div>
    <div class="item"><span class="checkbox"></span> 签名笔 ×2支</div>
    <div class="item"><span class="checkbox"></span> 老师伴手礼 ×2-3</div>
  </div>

  <div class="group">
    <h3><span class="icon">📷</span> 拍照 & 住宿</h3>
    <div class="item"><span class="checkbox"></span> 摄影师确认 / 横幅（可选）</div>
    <div class="item"><span class="checkbox"></span> 酒店预订确认 / 房型分配表</div>
  </div>

  <div class="group">
    <h3><span class="icon">🚑</span> 现场应急</h3>
    <div class="item"><span class="checkbox"></span> 签到表 ×2（打印） + 签到笔 ×3</div>
    <div class="item"><span class="checkbox"></span> 透明胶带 / 剪刀</div>
    <div class="item"><span class="checkbox"></span> 充电宝 ×1（备用）</div>
    <div class="item"><span class="checkbox"></span> 创可贴 / 防暑药（7月南京高温）</div>
  </div>

  <div class="note">
    ✅ 最后检查日：7月15日（聚会前3天）<br>
    ✅ 所有预订项目提前电话再确认一次<br>
    ✅ 应急备用金：100元/人，未用退还或群发红包
  </div>
</div>
</body>
</html>
```

- [ ] **Step 2: 预览**

Open `printables/checklist.html` in browser.
Verify: Green checkboxes, 5 groups, A4 portrait layout.

- [ ] **Step 3: Commit**

```bash
git add -f printables/checklist.html
git commit -m "feat: add printable preparation checklist with checkboxes

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 7: 配置趣味问答（问卷星）

**Files:**
- Reference: `materials/quiz-questions.md` (existing content)
- Reference: `content.json:180-234` (quiz data)

- [ ] **Step 1: 打开问卷星并创建问卷**

1. 打开 https://www.wjx.cn → 注册/登录
2. 点击「创建问卷」→「空白问卷」
3. 标题：「02交通毕业20周年 · 趣味问答」
4. 说明：「10道题 · 测测你还记得多少大学时光 · 每题1分」

- [ ] **Step 2: 逐题录入（10题）**

Create questions as single-choice or fill-in-blank:

```
Q1. [单选] 当年教《交通工程》的专业课老师叫什么名字？
    A. ___  B. ___  C. ___  D. ___

Q2. [单选] 大学四年，大家最常去的校外聚餐地点是哪里？
    A. ___  B. ___  C. ___  D. ___

Q3. [单选] 大二那年的班级活动/郊游去了哪里？
    A. ___  B. ___  C. ___  D. ___

Q4. [单选] 咱们班谁四年从来没有翘过课？（或者谁是翘课冠军？）
    A. ___  B. ___  C. ___  D. ___

Q5. [单选] 当年宿舍晚上几点熄灯断网？
    A. 22:30  B. 23:00  C. 23:30  D. 00:00

Q6. [单选] 食堂里大家公认最难吃/最好吃的一道菜是什么？
    A. ___  B. ___  C. ___  D. ___

Q7. [单选] 毕业散伙饭在哪家餐厅吃的？
    A. ___  B. ___  C. ___  D. ___

Q8. [填空] 咱们班一共多少人？几个宿舍？男女生比例是多少？

Q9. [单选] 当年学生会/社团里最活跃的是哪位同学？
    A. ___  B. ___  C. ___  D. ___

Q10. [填空] 请用一句话描述你对交通工程这个专业的理解。
```

- [ ] **Step 3: 设置计分规则**

- 每道单选题答对+1分，答错0分
- Q8填空设为手动阅卷（因为答案可能不同格式）
- Q10彩蛋设为不积分（因为答案无对错）
- 总分：8分（Q1-Q7 + Q9）
- 开启"答题后显示分数和排名"

- [ ] **Step 4: 生成二维码**

- 问卷设置中获取分享链接
- 用问卷星内置工具或 https://cli.im 生成二维码图片
- 保存二维码到 `printables/quiz-qrcode.png`
- 打印后贴在晚宴现场

- [ ] **Step 5: 在晚宴证书页添加二维码**

Update `award-certificates.html` or create a separate QR code print page.

- [ ] **Step 6: Commit**

```bash
git add -f printables/quiz-qrcode.png
git commit -m "feat: add quiz QR code for WeChat questionnaire

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 8: 更新网站链接到可打印物料

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 在网站导航或页脚添加物料下载链接**

Add a section to `index.html` with links to printable materials:

```html
<!-- Add after the existing footer section -->
<section id="downloads" style="padding:60px 20px; text-align:center; background:#fafaf7;">
  <h2 style="font-size:24px;color:#333;letter-spacing:3px;margin-bottom:8px;">配套物料下载</h2>
  <p style="color:#888;font-size:14px;margin-bottom:30px;">点击打印 | 建议用Chrome浏览器打开后 Ctrl+P</p>
  <div style="display:flex;flex-wrap:wrap;gap:16px;justify-content:center;max-width:800px;margin:0 auto;">
    <a href="printables/invitation.html" target="_blank" style="padding:14px 24px;background:linear-gradient(135deg,#ff6b6b,#ffa94d);color:white;border-radius:28px;text-decoration:none;font-weight:600;font-size:14px;">📧 邀请函</a>
    <a href="printables/itinerary-card.html" target="_blank" style="padding:14px 24px;background:linear-gradient(135deg,#ff6b6b,#ffa94d);color:white;border-radius:28px;text-decoration:none;font-weight:600;font-size:14px;">📋 行程单</a>
    <a href="printables/sign-in-sheet.html" target="_blank" style="padding:14px 24px;background:#059669;color:white;border-radius:28px;text-decoration:none;font-weight:600;font-size:14px;">✍️ 签到表</a>
    <a href="printables/award-certificates.html" target="_blank" style="padding:14px 24px;background:#059669;color:white;border-radius:28px;text-decoration:none;font-weight:600;font-size:14px;">🏆 颁奖证书</a>
    <a href="printables/checklist.html" target="_blank" style="padding:14px 24px;background:white;color:#333;border:1.5px solid #ddd;border-radius:28px;text-decoration:none;font-weight:600;font-size:14px;">📦 筹备清单</a>
  </div>
  <p style="color:#aaa;font-size:11px;margin-top:20px;">打开后按 Ctrl+P 打印 | 建议使用A4纸张 | Chrome打印效果最佳</p>
</section>
```

- [ ] **Step 2: 验证**

Reload `index.html` in browser. Verify downloads section renders with correct links.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add printable materials download section to website

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 9: 最终验证与整理

- [ ] **Step 1: 检查所有文件**

```bash
ls -la printables/
ls -la generated_images_qwen/
```

Expected output:
```
printables/
  invitation.html
  sign-in-sheet.html
  award-certificates.html
  itinerary-card.html
  checklist.html

generated_images_qwen/
  tee_front.png
  tee_back.png
  mug_front.png
  mug_back.png
```

- [ ] **Step 2: 逐个浏览器测试**

Open each HTML file in Chrome:
1. `printables/invitation.html` → Ctrl+P → should show 2 A5 pages
2. `printables/sign-in-sheet.html` → Ctrl+P → A4 landscape, 6×3 grid
3. `printables/award-certificates.html` → Ctrl+P → 6 A4 landscape certificates
4. `printables/itinerary-card.html` → Ctrl+P → A4 landscape, 3 equal columns
5. `printables/checklist.html` → Ctrl+P → A4 portrait, 5 groups

- [ ] **Step 3: 最终 commit**

```bash
git add -f printables/
git commit -m "chore: finalize all printable materials for reunion

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Summary

| Task | Output | Est. Time |
|------|--------|-----------|
| 1 | AI产品效果图 ×4 | 10 min |
| 2 | 邀请函 HTML | 10 min |
| 3 | 签到表 HTML | 10 min |
| 4 | 颁奖证书 HTML | 10 min |
| 5 | 行程单 HTML | 10 min |
| 6 | 筹备清单 HTML | 10 min |
| 7 | 趣味问答 问卷星 | 20 min |
| 8 | 网站下载入口 | 10 min |
| 9 | 最终验证 | 10 min |
| **Total** | | **~100 min** |

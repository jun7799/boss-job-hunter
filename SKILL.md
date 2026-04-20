---
name: boss-job-hunter
description: "Boss直聘职位搜索、智能匹配与飞书多维表格收录。自动登录用户Chrome搜索职位，根据简历匹配推荐，一键存入飞书表格。触发词：搜职位、找工作、Boss直聘、招聘搜索、职位匹配、投简历。"
version: 1.0.0
metadata:
  openclaw:
    emoji: "💼"
    requires:
      skills: ["web-access"]
---

# Boss直聘职位猎手

## 概述
通过 web-access CDP Proxy 直连用户 Chrome（携带 Boss 登录态），搜索指定城市和关键词的职位，根据用户简历智能匹配打分，将结果收录到飞书多维表格。

## 前置条件
1. **web-access CDP Proxy 运行中**：先执行 `bash {web-access-dir}/scripts/check-deps.sh` 确认
2. **用户 Chrome 已登录 Boss 直聘**：如果未登录，提示用户先登录
3. **用户简历信息**：从 `MEMORY.md` 或昨日 memory 文件获取

## 完整流程

### Step 1: 前置检查
```bash
bash "{web-access-dir}/scripts/check-deps.sh"
```
- 确认 Node.js 和 Chrome CDP 连接正常
- 如果 Proxy 未运行会自动启动

### Step 2: 搜索职位

#### 2.1 确定搜索参数
- **关键词**：从用户请求中提取（默认"AI产品经理"）
- **城市列表**：默认广州+深圳，用户可能指定其他城市
- **城市代码映射**（常见）：

| 城市 | 代码 |
|------|------|
| 北京 | 101010100 |
| 上海 | 101020100 |
| 广州 | 101280100 |
| 深圳 | 101280600 |
| 杭州 | 101210100 |
| 惠州 | 101280300 |
| 东莞 | 101281600 |
| 佛山 | 101280800 |
| 珠海 | 101280700 |
| 成都 | 101270100 |
| 武汉 | 101200100 |
| 长沙 | 101250100 |

#### 2.2 Boss 直聘搜索操作（每个城市重复）

**⚠️ 关键：Boss 直聘 URL 中 city 参数无效，必须通过页面内操作切换城市！**

```bash
# 1. 打开 Boss 直聘
curl -s "http://localhost:3456/new?url=https://www.zhipin.com"

# 2. 等待加载
sleep 3

# 3. 点击当前城市标签，打开城市选择器
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -d "(() => { document.querySelector('.city-label.active')?.click(); return 'ok'; })()"

# 4. 从热门城市列表选择目标城市（用 charCode 避免编码问题）
# 广州=24191,24030  深圳=28145,22323  北京=21271,20140  上海=19978,28023  杭州=26477,24030
sleep 1
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -d "(() => { const items = document.querySelectorAll('.city-list-hot li'); for (const li of items) { const t = li.innerText.trim(); if (t.charCodeAt(0) === {CITY_CHAR1} && t.charCodeAt(1) === {CITY_CHAR2}) { li.click(); return 'clicked'; } } return 'not found'; })()"

# 5. 城市切换完成后，输入搜索词
sleep 2
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -d "(() => { const input = document.querySelector('input.input'); input.focus(); const s = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set; s.call(input, '{KEYWORD}'); input.dispatchEvent(new Event('input', {bubbles:true})); input.dispatchEvent(new Event('compositionend', {bubbles:true})); document.querySelector('.search-btn').click(); return 'searched'; })()"

# 6. 等待搜索结果加载
sleep 4
```

#### 2.3 提取职位数据

**同时提取链接和文本信息，后续合并：**

```bash
# 提取链接
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -d "(() => { const links = Array.from(document.querySelectorAll('a[href*=job_detail]')).filter(a => !a.innerText.includes('查看更多') && !a.innerText.includes('职位搜索')); return JSON.stringify(links.map(a => ({title: a.innerText.substring(0,80), url: a.href}))); })()"

# 提取文本详情（薪资、经验、学历、公司、地区）
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -d "document.body.innerText.substring(0, 5000)"
```

#### 2.4 切换下一个城市
```bash
# 用 JS 直接修改 URL 中的城市参数（搜索成功后有效）
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -d "(() => { location.href = location.href.replace(/city=\\d+/, 'city={NEXT_CITY_CODE}'); return 'ok'; })()"
sleep 3
# 然后重新提取数据
```

#### 2.5 关闭后台 tab
```bash
curl -s "http://localhost:3456/close?target={TARGET_ID}"
```

### Step 3: 智能匹配打分

根据用户简历，对每个职位打推荐指数（1-5）：

| 分数 | 标准 |
|------|------|
| ⭐⭐⭐⭐⭐ 5分 | 经验要求匹配（5-10年/10年以上）+ 方向高度对口（AI Agent/AI Native/AI提效）+ 薪资好 |
| ⭐⭐⭐⭐ 4分 | 经验匹配 + 方向相关（AI产品/智能体/B端）或 方向对口但经验要求偏低 |
| ⭐⭐⭐ 3分 | 方向相关但经验不匹配，或薪资偏低 |
| ⭐⭐ 2分 | 薪资明显偏低或职位级别不匹配 |
| ⭐ 1分 | 不相关 |

**用户简历关键信息（从 memory 获取）：**
- AI应用产品经理，10年B端经验
- 亮点：5000万+销售额、AI原生开发能力、独立开发多个AI产品
- 技能：OpenClaw、Prompt Engineering、Agent 框架、Cursor 等 AI 工具

### Step 4: 收录到飞书多维表格

#### 4.1 创建或复用多维表格

**新表格：**
```
feishu_bitable_app.create(name: "Boss直聘-{关键词}职位")
```

**字段结构（create 时一次性定义，或逐步创建）：**

| 字段名 | 类型 | type值 |
|--------|------|--------|
| 职位名称 | 文本 | 1 |
| 城市 | 单选 | 3 |
| 薪资 | 文本 | 1 |
| 经验要求 | 文本 | 1 |
| 学历要求 | 文本 | 1 |
| 公司 | 文本 | 1 |
| 地区 | 文本 | 1 |
| 职位链接 | 超链接 | 15 |
| 推荐指数 | 数字 | 2 |

#### 4.2 删除默认空行
```
feishu_bitable_app_table_record.list → 获取空行 record_ids
feishu_bitable_app_table_record.batch_delete → 删除
```

#### 4.3 批量导入数据
```
feishu_bitable_app_table_record.batch_create(
  records: [
    {
      fields: {
        "职位名称": "AI产品经理",
        "城市": "广州",
        "薪资": "30-50K·16薪",
        "经验要求": "5-10年",
        "学历要求": "本科",
        "公司": "XXX科技",
        "地区": "南山区·科技园",
        "职位链接": {"link": "https://www.zhipin.com/job_detail/xxx.html", "text": "查看职位"},
        "推荐指数": 5
      }
    },
    ...
  ]
)
```

**注意：批量上限 500 条，同表不支持并发写。**

#### 4.4 输出结果
告知用户：
- 多维表格链接
- 各城市职位数量
- 推荐指数 TOP 3 的职位

## 变量参考

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `{web-access-dir}` | web-access skill 路径 | `~/.openclaw/workspace/skills/web-access` |
| `{TARGET_ID}` | CDP Proxy 创建的 tab targetId | 动态获取 |
| `{KEYWORD}` | 搜索关键词 | 用户指定 |
| `{CITY_CHAR1}`, `{CITY_CHAR2}` | 城市名前两个字符的 charCode | 见城市映射表 |

## 常见问题

### Q: Chrome CDP 未连接
提示用户：打开 `chrome://inspect/#remote-debugging`，勾选 "Allow remote debugging"，重启 Chrome。

### Q: Boss 直聘未登录
提示用户：在 Chrome 中登录 zhipin.com 后告知继续。

### Q: 搜索词乱码
使用 `String.fromCharCode()` 在 JS 中拼接中文，而非直接在 curl -d 中传中文。

### Q: Boss 拦截自动化浏览器
确保使用 web-access CDP Proxy 直连用户 Chrome，不要用 OpenClaw 内置浏览器（会被检测）。

### Q: 城市切换失败
Boss 会记住用户定位城市。如果 URL 参数无效，回退到页面内操作：点击城市标签 → 选择城市 → 搜索。

## 已有飞书表格

| 表格名 | 链接 | 说明 |
|--------|------|------|
| Boss直聘-AI产品经理职位 | https://e5zve6mvyq.feishu.cn/base/N5Exbiea2awDlzsRch6cItMLnuh | 2026-04-03 创建，广州+深圳 |

---
_Last updated: 2026-04-03_

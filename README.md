# 💼 Boss直聘职位猎手 - OpenClaw Skill

Boss直聘职位搜索、智能匹配与飞书多维表格收录的 OpenClaw Skill。

## 功能

- 🔍 **多城市职位搜索**：通过 CDP Proxy 直连用户 Chrome，搜索指定城市的 Boss 直聘职位
- 🎯 **智能匹配打分**：根据用户简历自动给职位打推荐指数（1-5 分）
- 📊 **飞书多维表格收录**：一键将职位数据批量导入飞书多维表格，包含职位名称、薪资、公司、推荐指数等

## 前置依赖

- [OpenClaw](https://github.com/openclaw/openclaw)
- [web-access](https://github.com/openclaw/openclaw) Skill（CDP Proxy）
- Chrome 浏览器已登录 Boss 直聘

## 安装

将 `SKILL.md` 放入 OpenClaw workspace 的 `skills/boss-job-hunter/` 目录即可。

## 使用

在 OpenClaw 对话中触发：

- "帮我搜一下广州的 AI 产品经理职位"
- "找工作，深圳+杭州，关键词产品总监"
- "Boss直聘搜职位"

## 支持城市

北京、上海、广州、深圳、杭州、惠州、东莞、佛山、珠海、成都、武汉、长沙等

## License

MIT

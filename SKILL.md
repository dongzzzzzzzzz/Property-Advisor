---
name: property-advisor
description: |
  房产搜索、决策与发布执行 skill。遇到租房、买房、找房、筛房、比较房源、通勤、周边配套、安全、区域判断，或发布出售/出租房源等请求时触发。
  这是一个 OpenClaw 优先的编排型 skill：C 端主动调用 ok-core-skill / gt-core-skill 获取房源并做地图增强；B 端解析发布资料，补齐关键字段后调用 ok-core-skill 发布房源。
metadata:
  short-description: 房产搜索与地图增强编排
---

# Property Advisor

## 角色

你不是“只看文档的决策层”。

你是一个执行型房产编排 skill，必须把请求推进到可交付结果，而不是只描述应该怎么做。

## 必走主链路

### C 端找房

当用户提出搜索、筛选、比较房源，或要求看通勤、周边、安全、区域时，默认执行下面的链路：

1. 识别用户需求和最小约束
2. 先判断请求应走 `ok-core-skill` 还是 `gt-core-skill`
3. 对所选上游 skill 做 preflight
4. 用上游 skill 获取房源列表
4. 对优先候选补齐详情页
5. 保存原始房源快照
6. 显式调用仓库内 `public-osm-map-context-skill/scripts/cli.py`
7. 基于房源原始信息和地图 assessments 生成结论
8. 输出固定 8 列候选表

不要把这条链路交给模型自由发挥，也不要只停在“我建议你去调用某个 skill”。

### B 端发布房源

当用户提出发布、出租、出售自己的房源，或发房产广告时，默认执行发布链路：

1. 判断是 `business_publish` 还是仍属于 C 端找房
2. 判断 `mode=sale|rent`
3. 复用市场路由：英国 / UK postcode / Gumtree 信号走 `gt-core-skill publish-listing`，非英国默认走 `ok-core-skill publish-property`
4. 抽取目标站点、房源类型、位置、价格、联系方式、图片路径、房型面积等字段
5. 抽取房源特色与强项；有位置、地址或 postcode 时必须调用仓库内地图 skill
6. 由 CLI 基于用户字段和地图 assessments 生成标题/描述，不允许模型自由补广告文案
7. 如果缺少必填字段，返回 `readiness_status=blocked_required`、`missing_fields` 和 `contextual_follow_up_questions`，禁止调用发布命令
8. 字段够但信息偏薄时返回 `readiness_status=thin_but_publishable`，允许 dry-run/填表并继续追问推荐补充项
9. 不带 `--submit`/`--confirm-submit` 时只填写表单或 dry-run；只有用户明确确认后才允许真实提交

B 端发布不得编造价格、面积、地址、联系方式、图片、房间数、卫浴数或未提供的设施。
B 端发布还不得编造 Dubai Mall、Burj Khalifa、rooftop pool、gym、No pets、security、wardrobes、metro 距离等未由用户提供或地图 assessments 证明的信息。

## 运行时硬规则

### 市场路由

- 默认 `market=auto`
- 英国相关请求默认走 `gt-core-skill`
- 非英国请求继续走 `ok-core-skill`
- 如果同时出现英国和非英国高置信地理信号，停止自动路由，明确要求用户确认市场

### ok-core-skill

路径发现顺序固定为：

1. `OK_CORE_SKILL_ROOT`
2. `PROPERTY_OK_SKILL_ROOT`
3. `/Users/a58/Desktop/skills/ok-core-skill`
4. 历史兼容路径 `/Users/a58/Desktop/ok-core-skill/skills/ok-core-skill`

执行顺序固定为：

1. `uv run python scripts/cli.py`
2. `ok-core-skill/.venv/bin/python scripts/cli.py`

禁止：

- 用裸 `python3 scripts/cli.py` 调 `ok-core-skill`
- 假设历史路径一定存在
- 不做 preflight 就直接开跑

发布补充规则：

- 发布房源调用 `publish-property`
- 必须显式传 `--country` 或 `--subdomain`
- 出租默认 `--rental-type entire`，租金周期缺失时默认 `--rent-period month`
- 图片必须是本地绝对路径；OpenClaw 上传附件需要先落地为本地文件
- 未明确确认真实发布时禁止传 `--submit`
- 中文 `平/平方米/㎡` 输入需换算为 OK 表单使用的 sqft，同时在报告中保留原始 sqm 和换算说明

### gt-core-skill

路径发现顺序固定为：

1. `GT_CORE_SKILL_ROOT`
2. `PROPERTY_GT_SKILL_ROOT`
3. 当前工作区 `.agents/skills` 与 `skills`
4. `$CODEX_HOME/skills` / `~/.codex/skills`
5. 本地桌面 `gt-core-skill` 开发路径

运行规则固定为：

1. 优先使用支持 `search + detail` 的 Bridge 版 Gumtree skill
2. Bridge 版优先 `uv run python scripts/cli.py`
3. Bridge 版失败时回退 `.venv/bin/python scripts/cli.py`
4. 只有找不到 Bridge 版时，才回退 API `search-listings` 模式

补充规则：

- `gt-core-skill` 的 `logged_in=false` 只记 warning，不算 preflight 失败
- GT API 版不支持详情补全时，必须把缺口写进 `缺失/未知`
- 英国请求默认仍然要补详情，不允许只停在搜索列表
- 英国发布房源必须自动走 `gt-core-skill publish-listing`
- GT 发布不确认时调用 `publish-listing --dry-run`
- GT 真实发布必须用户明确确认，并依赖 Gumtree session 中的 `publish_endpoint` 或显式 `--publish-endpoint`；缺失时报告配置错误，不得伪造成功

### 地图 skill

- 不依赖 nested skill 自动发现
- 直接调用当前仓库内的 `public-osm-map-context-skill/scripts/cli.py`
- 搜索类场景默认自动跑地图增强
- 发布类场景只要有 location / address / postcode，也默认自动跑地图增强
- 地图失败时继续给出候选表，但状态必须写成 `待补地图` 或 `待人工复核`
- 发布地图失败时继续生成发布报告，但标题/描述不得加入任何地图结论

## 何时必须跑地图增强

只要请求涉及以下任一项，就必须跑地图：

- 位置
- 通勤
- 地铁 / 公交 / tram / train
- 周边配套
- 生活便利度
- 安全
- 噪音
- 区域是否成熟

如果是标准找房搜索，即使用户没有显式强调，也默认首轮跑地图。

## 最终输出硬规则

默认输出固定 8 列：

1. `候选房源`
2. `状态`
3. `价格`
4. `位置`
5. `已满足`
6. `缺失/未知`
7. `淘汰原因/风险`
8. `房源链接`

补充规则：

- `房源链接` 只能用房源原帖链接，不能用 Google Maps / OSM 链接代替
- 可以额外保留地图复核链接，但不能替代原帖链接
- 没有原帖链接的候选不能点名展示，只能汇总进 `hidden_candidates`
- `已满足` 必须由房源原始信息 + 地图 assessments 共同生成
- `缺失/未知` 必须显式列出面积、卫浴、地图降级、路线未验证等缺口
- `淘汰原因/风险` 必须可回溯到价格异常、地图风险、信息不完整、预算不符等事实

## 地图 assessments 消费规则

每套房固定消费 4 个维度：

- `transport_access`
- `daily_convenience`
- `environment_risk`
- `area_maturity`

每个维度都必须带：

- `conclusion`
- `evidence`
- `confidence`
- `limitations`

不要把地图原始 POI 统计直接平铺给用户。
必须把地图数据转成“结论 + 证据”。

## 降级规则

### `precision=address`

- 可以作为首轮排序证据
- 也必须保留 `straight_line_estimate` 的边界说明

### `precision=area`

- 只能做片区判断
- 不能写成楼栋级精确距离
- 状态通常应为 `待人工复核`

### `precision=missing` 或 `status=degraded`

- 不能输出精确交通/配套断言
- 必须保留地图复核链接
- 候选表仍然要出，只是状态降级

## 调试入口

本仓库提供根级 CLI 供调试和回归使用：

```bash
python3 scripts/cli.py doctor --skip-browser-smoke
python3 scripts/cli.py route --query-text "我要出租 Dubai Marina 1BR"
python3 scripts/cli.py publish --query-text "我要出租 Dubai Marina 1BR furnished apartment near metro" --country uae --price 8000 --phone 501234567 --image "/absolute/path/photo.jpg" --dry-run
python3 scripts/cli.py search --keyword "southbank apartment" --city melbourne --country australia
python3 scripts/cli.py search --keyword "studio flat" --query-text "找 London 的 studio flat" --market auto
```

OpenClaw 是正式入口，但本地 CLI 是排查执行层问题的唯一推荐调试入口。

## 参考文档

- [references/data-contract.md](references/data-contract.md)
- [references/map-context-contract.md](references/map-context-contract.md)
- [references/output-contract.md](references/output-contract.md)
- [references/response-examples.md](references/response-examples.md)

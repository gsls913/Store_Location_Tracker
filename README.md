# 量贩零食门店地理数据跟踪

这是一个用于投研分析的门店地理另类数据项目。它通过地图平台返回的门店点位，跟踪量贩零食与新鲜零食品牌在不同城市的空间分布、竞争密度和选址环境，并生成一个可发布的静态地图网站。

项目当前的核心产品是城市级门店地图网站，而不是全国每日全量数据库。全国全量扫描在百度免费/企业日额度下成本过高，因此目前采用“重点城市精扫 + 其他城市粗扫/省级覆盖”的策略。

公开网站由 GitHub + Netlify 自动部署，Netlify 项目名为 `store-location-tracker`。

## 研究对象

量贩零食品牌：

- `万辰集团量贩零食`：好想来、老婆大人、来优品、吖嘀吖嘀、陆小馋等
- `鸣鸣很忙`：零食很忙、赵一鸣零食、鸣鸣很忙等

新鲜零食品牌：

- 一栗nutco
- 几多全
- 金粒门
- 蒲妈妈
- 久食山

网站上的“门店点位”指地图平台中的 POI，也就是地图数据库记录的门店兴趣点。它不是公司官方经营数据。

## 当前网站能力

网站入口：

- 本地生成入口：`site/index.html`
- 部署仓库入口：`github_deploy_repo/site/index.html`
- 生成源入口：`analytics/city_store_maps.html`

核心功能：

- 首页省份地图：点击省份后进入省内城市分块，已获取城市可继续进入城市地图。
- 城市地图：展示量贩零食和新鲜零食门店点位。
- 门店图层：按集团/子品牌控制显示，量贩零食始终排在新鲜零食前面。
- 极近门店 Top 10：
  - 不同品牌量贩零食最近 Top 10
  - 同一品牌量贩零食最近 Top 10
  - 新鲜零食 × 量贩零食最近 Top 10
- 环境图层：来自 OpenStreetMap Overpass 的商业设施点位、城市功能网格等。
- 扫描口径标识：每个城市标注为 `完整扫描`、`粗扫描` 或省级 `混合/粗扫汇总`。
- 制作人标识：`Created by Hammon`

## 数据口径

### 完整扫描

完整扫描表示这个城市在城市/区县粗扫基础上，进一步做过更细颗粒度的街道或街镇级补扫。它仍不是绝对无遗漏，但比粗扫描更适合看城市内门店总量、区县分布和竞争密度。

当前完整扫描城市：

- 上海市：较细颗粒度 Suggestion 扫描
- 长沙市：区县级 + 街镇级补扫
- 兰州市：街镇级补扫
- 深圳市：城市级 + 区县级粗扫基础上，完成 `1554 / 1554` 个街道级补扫请求

### 粗扫描

粗扫描主要使用城市级 + 区县级低额度关键词探测。它适合看大致空间分布和方向性样本，不适合把门店数量当作完整口径。

粗扫描在高密度城市会明显低估。长沙对比验证中，低额度方法只找到了约 205 个量贩点位，而完整旧方法有约 625 个，召回率约三分之一。这也是后来在网站里标注扫描口径的原因。

### 省级汇总

省级汇总页通常是混合口径：

- 湖南省：包含长沙完整扫描，也包含其他城市粗扫描。
- 广东省：多数城市为粗扫描，但深圳已经用完整扫描结果覆盖旧深圳粗扫数据。

跨城市和跨省比较时，必须先看扫描口径。粗扫描城市的“低门店数”可能只是漏抓，并不一定代表真实门店少。

## 数据获取方法

当前主要使用百度地图 Suggestion API。早期做过高德地图和百度地图的全国化流程探索，但当前城市网站以百度地图数据为主。

典型扫描层级：

1. 城市级探测：例如 `好想来`、`零食很忙深圳`。
2. 区县级探测：例如 `好想来(宝安区`、`好想来(深圳市宝安区`。
3. 街道/街镇级补扫：例如 `好想来(西乡`、`好想来(深圳西乡`、`好想来(宝安区西乡`。

为什么需要街道级补扫：

- 百度 Suggestion 接口每个 query 返回结果有限。
- 同一城市里门店密度高时，城市级或区县级 query 会被热门区域占满，远端街镇门店容易漏掉。
- 街道级拆分可以显著提高召回，但请求量会大幅增加。

请求额度原则：

- 每发起一次百度 Suggestion API 请求，就消耗一次百度请求额度。
- 不要随意启动全国或省级大范围扫描。
- 开始长扫描前，应先 dry-run 或估算请求量。
- 长扫描必须设置 `--max-requests` 和命令超时，方便额度耗尽时续跑。

## 去重与字段

主要门店字段：

- `uid`：百度地图 POI 唯一 ID
- `group`：集团，如 `鸣鸣很忙`、`万辰集团量贩零食`、`新鲜零食`
- `subbrand`：子品牌，如 `好想来`、`零食很忙`
- `name`：门店名
- `province`、`city`、`district`
- `address`
- `lat`、`lng`：百度 BD-09 坐标
- `business_hours`、`rating`：如果百度返回则保留
- `first_query`、`query_count`

地图展示时会把百度 BD-09 坐标转换为 WGS84 后绘制在 OpenStreetMap 底图上。

基础去重逻辑：

- 优先按 `uid` 合并。
- 对相同集团、子品牌、省市区县、地址的样本做同地址去重。
- 同地址重复时优先保留名称更完整、包含“零食”或子品牌名、区县信息更清楚的记录。

已知限制：

- 百度和高德覆盖不同，百度没有的点不一定代表门店不存在。
- 地图平台可能有同址重复 POI、命名不完整、坐标偏移或门店迁址滞后。
- 开店日期/POI 添加日期通常无法从百度 Suggestion 返回数据中直接获得。

## 环境图层

环境图层由 `scripts/build_environment_layers.py` 生成，数据源是 OpenStreetMap Overpass API。

当前保留：

- 商业设施点位
- 城市功能网格
- 门店附近商业设施距离
- 门店所在功能网格指标

已经移除：

- 人口密度代理图层
- 夜间活跃度代理图层

移除原因：这些代理值不是可靠真实数据，容易误导分析。未来除非找到可靠来源，不要重新加入。

OpenStreetMap 数据可能不完整，尤其在国内城市覆盖不均。环境图层适合作为辅助观察，不应作为严肃定量结论的唯一依据。

## 目录结构

关键目录和文件：

- `scripts/build_city_store_maps.py`：主网站生成器。城市配置、数据源、扫描口径、首页分组都在这里。
- `scripts/build_environment_layers.py`：环境图层生成器。
- `scripts/resume_*_suggestion.py`：各城市或省份的百度 Suggestion 扫描/续跑脚本。
- `diagnostics/`：门店抓取原始结果、合并结果、请求日志和诊断文件。
- `data/environment/`：环境图层 JSON。
- `analytics/`：生成出来的 HTML 页面。
- `site/`：本地静态网站目录。
- `github_deploy_repo/`：真正用于 GitHub/Netlify 部署的 Git 仓库。
- `github_deploy_repo/site/`：部署用静态网站目录。
- `poi_tracker/`：早期全国 POI 跟踪、SQLite、周度分析相关代码。当前城市网站主要不走这条主线，但不要随意删除。
- `archive/`：旧仪表盘、旧输出和参考资料。
- `AGENTS.md`：给 Codex 或未来 coding agent 的操作规范。
- `apikey.env`：本地 API key，永远不要提交、打印或发布。

注意：项目根目录不是部署仓库。部署仓库是 `github_deploy_repo/`。

## 常用工作流

### 重建网站

修改数据源、城市配置、生成器或环境图层后，运行：

```powershell
python scripts\build_city_store_maps.py
Copy-Item analytics\city_store_maps.html site\index.html -Force
Copy-Item analytics\city_maps\*.html site\city_maps\ -Force
Copy-Item analytics\province_maps\*.html site\province_maps\ -Force
Copy-Item analytics\city_store_maps.html github_deploy_repo\site\index.html -Force
Copy-Item analytics\city_maps\*.html github_deploy_repo\site\city_maps\ -Force
Copy-Item analytics\province_maps\*.html github_deploy_repo\site\province_maps\ -Force
```

### 验证

```powershell
python -m pytest -q --basetemp .tmp\pytest
```

检查生成页面里的 JavaScript 语法：

```powershell
@'
const fs = require('fs');
for (const file of ['site/index.html','site/city_maps/shenzhen.html','site/province_maps/guangdong.html']) {
  const html = fs.readFileSync(file, 'utf8');
  const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m => m[1]);
  for (const script of scripts) new Function(script);
  console.log('syntax ok', file, scripts.length);
}
'@ | node
```

### 发布

```powershell
cd github_deploy_repo
git status --short
git add site
git commit -m "Update city store maps"
git push origin main
```

Netlify 会自动读取 GitHub 最新提交并重新部署，网站 URL 不变。

如果只改了根目录 README，需要同步到部署仓库：

```powershell
Copy-Item README.md github_deploy_repo\README.md -Force
```

## 添加或更新一个城市

1. 先确定扫描策略：
   - 门店少/下沉城市：先用城市级 + 区县级低额度探测。
   - 高密度城市或重点样本：进一步做街道/街镇级补扫。
2. 估算请求量并确认额度。
3. 运行对应 `scripts/resume_*_suggestion.py` 脚本。
4. 检查 request log 是否有 `status=302 天配额超限`。
5. 检查输出 CSV 的门店数量、品牌结构、区县分布是否合理。
6. 必要时生成或刷新环境图层：

```powershell
python scripts\build_environment_layers.py --city <slug> --max-tiles 60 --max-seconds 900 --refresh
```

7. 在 `scripts/build_city_store_maps.py` 中加入或更新城市配置。
8. 明确城市扫描口径：
   - 完整扫描
   - 粗扫描
   - 混合/粗扫汇总
9. 重建、验证、同步部署目录。
10. 如果这是重要口径变化，更新本 README。

## 已有重点城市状态

截至 2026-06-28：

- 上海市：完整扫描
- 长沙市：完整扫描
- 兰州市：完整扫描
- 深圳市：完整扫描，量贩点位 489，新鲜零食点位 7
- 邵阳市：粗扫描
- 北京市：粗扫描
- 广州市：粗扫描
- 商丘市：粗扫描
- 周口市：粗扫描
- 湖南省汇总：混合口径
- 广东省汇总：混合口径，深圳已用完整扫描结果覆盖

这些状态以后如果改变，应同步更新 `scripts/build_city_store_maps.py` 的 `scan_profile()` 和本 README。

## 投研使用注意

- 完整扫描城市可以更认真地看城市内空间格局、区县分布、极近门店和商圈竞争。
- 粗扫描城市更适合做方向性观察，不适合直接比较绝对门店数。
- 新鲜零食门店目前数量少，且扫描深度通常低于量贩零食，应作为辅助变量。
- 地图点位不是流水、不是开店日期，也不是官方门店清单。
- 门店新增或消失应结合地图页面人工抽样核验。
- 对外展示时要保留扫描口径说明，避免把样本数据讲成完整事实。

## 维护约定

重要变化必须更新这个 README，包括但不限于：

- 新增城市或省份
- 某城市从粗扫描升级为完整扫描
- 重要数据源替换
- 品牌关键词变化
- 去重规则变化
- 网站功能和展示口径变化
- 发布流程变化

`AGENTS.md` 是给 Codex 的操作规范，`README.md` 是给人看的项目说明。两者都重要，但 README 应保持对项目整体逻辑的最新解释。

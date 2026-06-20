# 量贩零食门店另类数据跟踪

这个项目用地图 Web API 跟踪量贩零食品牌门店点位，形成可重复的门店快照、增量变化和区域分布数据。POI 指地图平台里的兴趣点；在本项目的用户界面里统一称为“门店点位”。

## 适合回答的问题

- 鸣鸣很忙、万辰系品牌的门店数量是否继续扩张？
- 哪些省、市、区县新增更快？
- 哪些门店从地图 POI 中消失，可能对应闭店或迁址？
- 同一行政区内不同品牌的覆盖强度如何变化？

## 快速开始

1. 安装依赖：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

2. 获取并设置高德 Web 服务 Key。具体步骤见 [docs/amap_key.md](docs/amap_key.md)。

```powershell
$env:AMAP_KEY="你的高德Web服务Key"
```

也可以把 Key 写到项目根目录的 `.env` 或 `apikey.env` 文件中：

```text
AMAP_KEY=你的高德Web服务Key
```

3. 检查 Key 和 API 连通性：

```powershell
python -m poi_tracker doctor
```

4. 先跑小样本冒烟测试：

```powershell
.\scripts\run_smoke.ps1
```

5. 拉取行政区 adcode。首次建议先用市级或少数省份试跑，确认额度和效果后再跑区县级全国数据：

```powershell
python -m poi_tracker fetch-adcodes --output data/adcodes_city.csv --level city
```

6. 抓取 POI 并写入 SQLite 快照：

```powershell
python -m poi_tracker crawl-amap --config config/brands.json --adcodes data/adcodes_city.csv --db data/poi_tracker.sqlite
```

7. 导出最新快照和变化：

```powershell
python -m poi_tracker export --db data/poi_tracker.sqlite --out-dir exports
```

也可以直接跑完整市级版本：

```powershell
.\scripts\run_full_city.ps1
```

如果使用免费额度，建议先看 [docs/quota_strategy.md](docs/quota_strategy.md)。每天跑全国全量通常不够，推荐生成每日轮转分片：

```powershell
.\scripts\setup_daily_shards.ps1
.\scripts\run_daily_shard.ps1 -ShardNumber 1
```

或者让脚本按日期自动选择 shard：

```powershell
.\scripts\run_today_shard.ps1
```

可选：注册 Windows 每日计划任务：

```powershell
.\scripts\register_daily_task.ps1 -Time "20:30"
```

百度地图企业额度更高时，推荐把百度作为主数据源。策略说明见 [docs/provider_strategy.md](docs/provider_strategy.md)。

```powershell
python -m poi_tracker doctor-baidu
.\scripts\setup_baidu_daily_shards.ps1
.\scripts\run_baidu_weekly_next_shard.ps1
```

如果 2000 次/日不足以跑完整全国，使用周度续跑：每次运行自动执行本周尚未完成的下一个 shard；一周内全部 shard 完成后，仪表盘才把该周作为正式周度快照。

生成正式分析指标和 HTML 仪表盘：

```powershell
python -m poi_tracker analyze --db data/poi_tracker.sqlite --out-dir analytics
```

打开 [analytics/dashboard.html](analytics/dashboard.html) 查看周度趋势、地图、区域分层、自动洞察、任务管理和数据质量面板。

正式使用手册见 [docs/investment_dashboard_manual.md](docs/investment_dashboard_manual.md)。建议长期使用前先阅读其中的周度快照口径、指标解释和数据质量边界。

如果要在页面里点击按钮续跑本周百度分片，需要启动本地服务：

```powershell
.\scripts\start_dashboard.ps1
```

然后打开：

```text
http://127.0.0.1:8765
```

## 配置说明

品牌配置在 [config/brands.json](config/brands.json)。每个品牌可以设置多个关键词和别名：

- `brand_id`：内部统一品牌 ID。
- `display_name`：显示名称。
- `keywords`：实际传给地图 API 的搜索词。
- `include_aliases`：用于二次过滤 POI 名称，降低误匹配。
- `exclude_keywords`：排除明显无关结果。

## 数据表

- `runs`：每次抓取任务。
- `pois`：去重后的 POI 主表。
- `observations`：每次 run 观测到的 POI 明细。
- `run_seen`：某次 run 看到过哪些 POI。

导出文件：

- `latest_pois.csv`：最新 run 的 POI 明细。
- `changes_since_previous.csv`：相对上一次 run 的新增、消失、持续存在。
- `brand_region_summary.csv`：按品牌、省、市、区县聚合。
- `searches.csv`：本次 run 实际搜索过的品牌、关键词、城市和请求计数。
- `all_known_pois.csv`：历史累计已知门店，适合分片日更后看全局最新已知状态。
- `all_known_brand_region_summary.csv`：历史累计已知门店的区域聚合。
- `run_history.csv`：抓取任务历史。
- `provider_region_comparison.csv`：高德/百度按品牌和区域的数量交叉对照。

## 注意事项

- POI 不是公司经营数据，新增或消失需要人工抽样核验。
- 地图平台有更新延迟，通常更适合做趋势跟踪，不适合精确到日。
- 如果某品牌单个区县搜索结果接近分页上限，需要继续拆分到街道、矩形网格或结合百度数据交叉验证。

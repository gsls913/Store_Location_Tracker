# 网站发布方式

## 推荐方式：GitHub + Netlify 自动部署

当前静态网站发布目录是 `site/`，Netlify 配置文件是 `netlify.toml`。

第一次发布：

1. 在 GitHub 新建一个仓库。
2. 在本地把本项目初始化为 Git 仓库并推送到 GitHub。
3. 打开 Netlify，选择 `Add new site` -> `Import an existing project`。
4. 选择刚才的 GitHub 仓库。
5. Netlify 会读取 `netlify.toml`，发布目录为 `site/`。
6. 部署完成后，Netlify 会给一个公开 URL。

以后更新：

1. 重新获取或整理门店数据。
2. 运行 `python scripts\build_city_store_maps.py` 生成 `analytics/` 页面。
3. 把 `analytics/city_store_maps.html` 复制为 `site/index.html`，把 `analytics/city_maps/*.html` 复制到 `site/city_maps/`。
4. 提交并推送 GitHub。
5. Netlify 会自动重新部署，URL 不变。

## 临时方式：Netlify Drop

如果只是临时发布，可以把 `publish_site_20260620_v3.zip` 拖到 Netlify Drop 页面。
这种方式最快，但之后每次更新都要重新上传 zip。


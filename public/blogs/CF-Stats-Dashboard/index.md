本文档记录了如何在基于 MkDocs 的静态站点中，利用 Cloudflare Workers 实现无服务器的实时流量24小时统计看板。

理论上其他 网站框架 也完全可以复用。只是我个人早期使用MkDocs时使用了这种方式

## 📌 技术实现原理

  * **架构**：部署在 Cloudflare Worker 上的单文件流量分析面板后端 (无需服务器/Docker)，配合静态 HTML 前端展示。
  * **限制说明**：**仅展示 24 小时内数据**，这是因为 Cloudflare GraphQL Analytics API 的免费调用限制。
  * **优势**：零成本、高响应速度、**完美融入静态博客系统**。

-----
## 第一步：获取 Cloudflare API 凭证 (变量与机密)

我们需要获取两个关键凭证，以便 Worker 有权限读取你的站点数据。

1.  **获取 `ZONE_ID` (区域 ID)**：
      * 登录 Cloudflare 控制台，进入你要统计的域名（如 `example.com`）的 **概述 (Overview)** 页面。
      * 向下滑动，在右侧边栏找到 **API** -\> **区域 ID (Zone ID)**，复制这段 32 位的字符。
2.  **获取 `API_TOKEN` (API 令牌)**：
      * 进入 Cloudflare 右上角[我的个人资料 -\> API 令牌](https://dash.cloudflare.com/profile/api-tokens)。
      * 点击 **创建自定义令牌**，命名随意。
      * **权限配置 (必须完全一致)**：
          * `区域 (Zone)` -\> `分析 (Analytics)` -\> `读取 (Read)`
          * `区域 (Zone)` -\> `区域 (Zone)` -\> `读取 (Read)`
      * **区域资源**：`包括` -\> `特定区域` -\> 选择你的域名。
      * 创建后，**立刻复制这串 Token**（关闭后无法再次查看）。

举例：
#### mkdocs-stats-api API 令牌摘要
此 API 令牌将影响以下帐户和区域，以及它们各自的权限
- 所有帐户 - 帐户设置:读取
- xxxx@qq.com 's Account
- - example.com - 区域设置:编辑, 区域:读取, DNS:读取, 清除缓存:清除, Analytics:读取

-----
## 第二步：部署 Cloudflare Worker (后端 API)

1.  在 Cloudflare 控制台左侧菜单进入 **Workers & Pages**，创建一个新的 Worker（例如命名为 `mkdocs-stats-api`）。
2.  进入 **设置 (Settings) -\> 变量和机密 (Variables and Secrets)**。
3.  添加两个**机密 (Secret)** 变量：
      * 名称：`ZONE_ID`，值：你的区域 ID。
      * 名称：`API_TOKEN`，值：你的 API 令牌。
4.  点击 **编辑代码**，填入以下后端数据接口逻辑，并部署：

```javascript
// Worker名称: CF-Stats-Dashboard-Pro
// 功能: 美化版 Cloudflare 流量分析面板 + API

export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // === API 路由 ===
    if (url.pathname === "/api/data") {
      return await handleApiRequest(request, env);
    }

    // === 前端页面路由 ===
    return await handleHtmlRequest();
  }
};

// --- 后端逻辑 ---
async function handleApiRequest(request, env) {
  const corsHeaders = {
    "Access-Control-Allow-Origin": "*", // 允许你的 MkDocs 站点访问
    "Access-Control-Allow-Methods": "GET",
    "Content-Type": "application/json",
    // 关键优化：添加 60秒 缓存，防止频繁刷新刷爆 API
    "Cache-Control": "public, max-age=60" 
  };

  try {
    // 逻辑：总量查当天，详情查过去 23 小时 (避开 86400s 限制)
    const now = new Date();
    const safeTimeAgo = new Date(now.getTime() - 23 * 60 * 60 * 1000).toISOString(); 
    const today = new Date().toISOString().split('T')[0];
    
    // GraphQL 查询
    const query = `
      query {
        viewer {
          zone_totals: zones(filter: {zoneTag: "${env.ZONE_ID}"}) {
            totals: httpRequests1dGroups(limit: 1, filter: {date_geq: "${today}"}) {
              sum { requests, bytes, threats, cachedRequests }
            }
          }
          zone_trends: zones(filter: {zoneTag: "${env.ZONE_ID}"}) {
            trends: httpRequests1hGroups(limit: 24, orderBy: [datetime_ASC], filter: {datetime_geq: "${safeTimeAgo}"}) {
              dimensions { datetime }
              sum { requests }
            }
          }
          zone_geo: zones(filter: {zoneTag: "${env.ZONE_ID}"}) {
            geo: httpRequestsAdaptiveGroups(limit: 10, orderBy: [count_DESC], filter: {datetime_geq: "${safeTimeAgo}"}) {
              dimensions { clientCountryName }
              count
            }
          }
        }
      }
    `;

    const cfResponse = await fetch("https://api.cloudflare.com/client/v4/graphql", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${env.API_TOKEN}`
      },
      body: JSON.stringify({ query })
    });

    const result = await cfResponse.json();
    if (!result.data) throw new Error(JSON.stringify(result.errors));

    const viewer = result.data.viewer;
    const totals = viewer.zone_totals?.[0]?.totals?.[0]?.sum || { requests: 0, bytes: 0, threats: 0, cachedRequests: 0 };
    
    const data = {
      summary: {
        total: totals.requests,
        cached: totals.cachedRequests,
        uncached: totals.requests - totals.cachedRequests,
        threats: totals.threats,
        bytes: totals.bytes
      },
      chart: {
        // 只返回简单数据，让前端去处理格式
        times: viewer.zone_trends?.[0]?.trends.map(t => t.dimensions.datetime) || [],
        values: viewer.zone_trends?.[0]?.trends.map(t => t.sum.requests) || []
      },
      // 返回国家代码 (如 CN)，让前端转中文
      geo: viewer.zone_geo?.[0]?.geo.map(g => ({ 
        code: g.dimensions.clientCountryName, 
        count: g.count 
      })) || []
    };

    return new Response(JSON.stringify(data), { headers: corsHeaders });
  } catch (err) {
    return new Response(JSON.stringify({ error: err.message }), { status: 500, headers: corsHeaders });
  }
}

// --- 前端页面 (引入 Tailwind CSS + Chart.js) (备用直接访问页) ---
async function handleHtmlRequest() {
  const html = `<!DOCTYPE html><html lang="zh-CN"><head><meta charset="UTF-8"><title>API 运行正常</title></head><body><h1>API is running. Access /api/data</h1></body></html>`;
  return new Response(html, { headers: { "Content-Type": "text/html;charset=UTF-8" } });
}
```

-----

## 第三步：创建前端可视化页面 (HTML)

在你的 MkDocs 项目的静态资源文件夹中（例如 `docs/assets/`），创建一个名为 `Cloudflare-mkdocs-stats.html` 的文件。

这个文件负责从 Worker API 拉取数据，并使用 ECharts 渲染出漂亮的半透明磨砂风格图表。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>website Stats</title>
    <script src="https://cdn.jsdelivr.net/npm/echarts@5.4.3/dist/echarts.min.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;800&family=ZCOOL+KuaiLe&display=swap" rel="stylesheet">

    <style>
        /* --- 全局重置 --- */
        body { margin: 0; padding: 20px; font-family: 'Nunito', sans-serif; background: transparent; color: #546e7a; }
        .cf-dashboard-container { max-width: 1200px; margin: 0 auto; }

        /* --- 顶部数据卡片网格 --- */
        .dashboard-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 20px; margin-bottom: 30px; }
        .stat-card {
            background: rgba(255, 255, 255, 0.8); backdrop-filter: blur(10px); padding: 20px; border-radius: 16px;
            text-align: center; border: 1px solid rgba(255, 255, 255, 0.6); box-shadow: 0 4px 15px rgba(0, 0, 0, 0.03);
            transition: all 0.3s ease; position: relative; overflow: hidden;
        }
        .stat-card:hover { transform: translateY(-5px); box-shadow: 0 8px 25px rgba(0, 230, 118, 0.15); border-color: #b9f6ca; }
        .stat-card::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 4px; background: linear-gradient(90deg, #a5d6a7, #81c784); opacity: 0.8; }
        .stat-card.pink-accent:hover { box-shadow: 0 8px 25px rgba(244, 143, 177, 0.2); border-color: #f8bbd0; }
        .stat-card.pink-accent::before { background: linear-gradient(90deg, #f48fb1, #f06292); }
        .stat-value { font-size: 2em; font-weight: 800; color: #2e7d32; margin: 10px 0 5px; line-height: 1; font-family: 'Nunito', sans-serif; }
        .stat-label { font-size: 0.9em; color: #78909c; font-weight: 600; }

        /* --- 图表容器 --- */
        .chart-wrapper { background: rgba(255, 255, 255, 0.6); border-radius: 16px; padding: 20px; margin-bottom: 25px; border: 1px solid rgba(255, 255, 255, 0.5); box-shadow: 0 4px 20px rgba(0,0,0,0.02); }
        .chart-box { width: 100%; height: 350px; }
        h3.cf-title { margin-top: 0; margin-bottom: 15px; font-size: 1.2em; font-family: 'ZCOOL KuaiLe', cursive; color: #455a64; display: flex; align-items: center; gap: 10px; }
        h3.cf-title::before { content: ''; display: inline-block; width: 6px; height: 20px; background: #66bb6a; border-radius: 3px; }

        /* --- 底部版权 --- */
        .credits { text-align: center; margin-top: 40px; font-size: 0.85em; color: #90a4ae; font-family: 'Nunito', sans-serif; opacity: 0.8; }
        .heart { color: #ec407a; display: inline-block; animation: beat 1.5s infinite; }
        @keyframes beat { 0%, 100% { transform: scale(1); } 50% { transform: scale(1.2); } }
    </style>
</head>
<body>

<div class="cf-dashboard-container">
    <div class="dashboard-grid">
        <div class="stat-card">
            <div class="stat-label">24H 请求总数</div>
            <div class="stat-value" id="val-requests">-</div>
        </div>
        <div class="stat-card">
            <div class="stat-label">产生流量</div>
            <div class="stat-value" id="val-traffic">-</div>
        </div>
        <div class="stat-card">
            <div class="stat-label">CDN 命中次数</div>
            <div class="stat-value" id="val-cached" style="color:#00b894">-</div>
        </div>
        <div class="stat-card pink-accent">
            <div class="stat-label">拦截威胁</div>
            <div class="stat-value" id="val-threats" style="color:#ef5350">-</div>
        </div>
    </div>

    <div class="chart-wrapper">
        <h3 class="cf-title">📈 请求数趋势 (Requests/Hour)</h3>
        <div id="chart-trends" class="chart-box"></div>
    </div>

    <div class="dashboard-grid" style="grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));">
        <div class="chart-wrapper"> 
            <h3 class="cf-title">🍩 请求类型分布</h3>
            <div id="chart-cache" class="chart-box" style="height: 300px;"></div>
        </div>
        <div class="chart-wrapper">
            <h3 class="cf-title">🌍 地区访问量前十</h3>
            <div id="chart-geo" class="chart-box" style="height: 300px;"></div>
        </div>
    </div>

    <div class="credits">
        Designed with <span class="heart">❤</span> by <strong>KanameMadoka520</strong> & <strong>LainElaina</strong><br>
        Powered by Cloudflare Workers & ECharts
    </div>
</div>

<script>
  // ⚠️ 替换为你自己的 Worker API 链接
  const WORKER_API_URL = 'https://mkdocs-stats-api.xxx123456.workers.dev/api/data';

  const regionNames = new Intl.DisplayNames(['zh-CN'], { type: 'region' });
  function getCountryName(code) { try { return regionNames.of(code); } catch { return code; } }

  function formatBytes(bytes) {
      if (!+bytes) return '0 B';
      const k = 1024; const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
      const i = Math.floor(Math.log(bytes) / Math.log(k));
      return `${parseFloat((bytes / Math.pow(k, i)).toFixed(2))} ${sizes[i]}`;
  }

  const trendChart = echarts.init(document.getElementById('chart-trends'));
  const cacheChart = echarts.init(document.getElementById('chart-cache'));
  const geoChart = echarts.init(document.getElementById('chart-geo'));

  window.addEventListener('resize', () => { trendChart.resize(); cacheChart.resize(); geoChart.resize(); });

  function fetchDashboardData() {
    trendChart.showLoading({ color: '#66bb6a' });
    cacheChart.showLoading({ color: '#66bb6a' });
    geoChart.showLoading({ color: '#66bb6a' });

    fetch(WORKER_API_URL)
      .then(res => res.json())
      .then(data => {
        if(data.error) throw new Error(data.error);

        document.getElementById('val-requests').innerText = data.summary.total.toLocaleString();
        document.getElementById('val-traffic').innerText = formatBytes(data.summary.bytes);
        document.getElementById('val-cached').innerText = data.summary.cached.toLocaleString();
        document.getElementById('val-threats').innerText = data.summary.threats;

        const textColor = '#546e7a';

        trendChart.hideLoading();
        trendChart.setOption({
          tooltip: { trigger: 'axis', backgroundColor: 'rgba(255,255,255,0.9)', borderColor: '#eee' },
          grid: { left: '3%', right: '4%', bottom: '3%', containLabel: true },
          xAxis: { type: 'category', boundaryGap: false, data: data.chart.times.map(t => new Date(t).getHours() + ':00'), axisLine: { lineStyle: { color: '#cfd8dc' } }, axisLabel: { color: textColor } },
          yAxis: { type: 'value', splitLine: { lineStyle: { type: 'dashed', color: '#eceff1' } }, axisLabel: { color: textColor } },
          series: [{ name: '请求数', type: 'line', smooth: true, symbol: 'none', lineStyle: { width: 3, color: '#66bb6a' }, areaStyle: { color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{ offset: 0, color: 'rgba(102, 187, 106, 0.5)' }, { offset: 1, color: 'rgba(102, 187, 106, 0.0)' }]) }, data: data.chart.values }]
        });

        cacheChart.hideLoading();
        cacheChart.setOption({
          tooltip: { trigger: 'item' }, legend: { bottom: '0', textStyle: { color: textColor } },
          series: [{ type: 'pie', radius: ['45%', '70%'], center: ['50%', '45%'], avoidLabelOverlap: false, itemStyle: { borderRadius: 10, borderColor: '#fff', borderWidth: 2 }, label: { show: false }, data: [{ value: data.summary.cached, name: 'CDN缓存 (命中)', itemStyle: { color: '#66bb6a' } }, { value: data.summary.uncached, name: '回源请求', itemStyle: { color: '#f06292' } }] }]
        });

        geoChart.hideLoading();
        const geoNames = data.geo.slice(0, 10).map(g => getCountryName(g.code));
        const geoValues = data.geo.slice(0, 10).map(g => g.count);
        geoChart.setOption({
          tooltip: { trigger: 'axis', axisPointer: { type: 'shadow' } }, grid: { left: '3%', right: '4%', bottom: '3%', containLabel: true }, xAxis: { type: 'value', show: false },
          yAxis: { type: 'category', data: geoNames.reverse(), axisLine: { show: false }, axisTick: { show: false }, axisLabel: { color: textColor, fontWeight: 'bold' } },
          series: [{ name: '访问量', type: 'bar', data: geoValues.reverse(), barWidth: '60%', itemStyle: { borderRadius: [0, 20, 20, 0], color: new echarts.graphic.LinearGradient(0, 0, 1, 0, [{ offset: 0, color: '#42a5f5' }, { offset: 1, color: '#26c6da' }]) }, label: { show: true, position: 'right', color: textColor } }]
        });
      })
      .catch(err => {
        console.error(err);
        document.getElementById('val-requests').innerText = "API Error";
        document.getElementById('val-requests').style.color = "#ef5350";
      });
  }

  document.addEventListener("DOMContentLoaded", fetchDashboardData);
</script>
</body>
</html>
```

-----

## 第四步：在 MkDocs Markdown 中嵌入

编辑你想要展示面板的 Markdown 文件（例如 `docs/update/Cloudflare-mkdocs-stats.md`），加入状态提示以及 iframe 容器即可。

```html
<div class="stats-header">
    <h1 class="stats-title">📊 站点访问统计 (24小时)</h1>
    <div class="stats-subtitle">(2026年2月1日15时添加的功能 - 测试版)</div>
</div>

<div class="network-alert">
    <div class="alert-icon">⚡</div>
    <div>
        <strong>网络状态提示：</strong><br>
        因为是实时请求 API，需要您当前网络对 <strong>Cloudflare</strong> 和 <strong>UptimeRobot</strong> 连接顺畅才会出结果。<br>
        <span style="opacity: 0.8; font-size: 0.9em;">(💡 如果开了魔法，应该很快就出结果！)</span>
    </div>
</div>

<div class="iframe-wrapper">
    <iframe class="stats-frame" title="Cloudflare 24 小时站点访问统计面板" src="../../assets/Cloudflare-mkdocs-stats.html" style="width: 100%; height: 950px; border: none; overflow: hidden;"></iframe>
</div>
```

-----

## 💡 进阶

如果这个 24 小时的面板不能满足你的需求，想要查询 **3 天、7 天甚至 30 天** 的历史数据，你需要突破免费 API 的限制。

**操作思路**：
自己用一台闲置电脑、树莓派或 VPS，跑 [Geekertao/cloudflare-analytics](https://www.google.com/search?q=https://github.com/Geekertao/cloudflare-analytics) Docker 项目，后台利用定时脚本不断拉取保存。然后通过 **Cloudflare Tunnel (Zero Trust)** 进行内网穿透。这样既能使用原项目所有功能（突破 24H 限制），又能利用 Cloudflare 网络进行访问保护。

-----
# JavaScript Zillow 抓取教程：不用写反爬逻辑也能稳定拿到房源数据的方法

## 为什么直接爬 Zillow 越来越难了

上个月我帮一个做北美房产数据分析的朋友写爬虫，目标很简单——把 Zillow 上某个 ZIP code 的在售房源价格、面积、地址批量拉下来。结果跑了不到 50 个请求，IP 就被封了。换了代理池，过了两天又被封。Zillow 的反爬机制这两年升级得很猛，光靠 puppeteer 加随机 User-Agent 已经扛不住了。

后来我换了个思路：把反爬这块完全交给专门干这事的 API 服务，自己只管写解析逻辑。折腾了一圈下来，确实省心不少。这篇文章就把我整个流程拆开讲，从环境搭建到代码实现，到踩过的坑，全给你过一遍。

---

**一句话结论**：用 ScraperAPI 处理 Zillow 的反爬层（IP 轮换、浏览器指纹、验证码绕过），你只需要写 JavaScript 解析逻辑就行，省下来的时间够你多分析三个城市的数据了。

👉 [立即获取 ScraperAPI 的 5000 次免费额度开始测试](https://www.scraperapi.com/?fp_ref=coupons)

---

## Zillow 反爬机制到底做了什么

Zillow 不是简单地看你请求频率。它的防护层至少包含这几个东西：

- **行为指纹检测**：检查你的 TLS 指纹、HTTP/2 握手特征、Canvas 渲染结果
- **动态渲染**：大量房源数据通过 JavaScript 异步加载，纯 HTTP 请求拿到的 HTML 是空壳
- **IP 信誉评分**：数据中心 IP 基本秒封，住宅代理也会被限速
- **验证码触发**：短时间内同一 session 请求过多会弹 reCAPTCHA

我之前用 Puppeteer 硬刚，光处理这些就写了 400 多行代码。维护成本太高了。

## ScraperAPI 怎么解决这些问题

ScraperAPI 本质上是一个代理 + 渲染引擎的中间层。你把目标 URL 丢给它，它帮你处理 IP 轮换、浏览器模拟、验证码识别，然后把渲染完成的 HTML 返回给你。

对 Zillow 这种重度依赖 JS 渲染的站点，它有个 `render=true` 参数，会用无头浏览器把页面完整渲染后再返回。这意味着你拿到的 HTML 里已经包含了所有动态加载的房源数据。

我用了大概三个月，跑 Zillow 的成功率稳定在 95% 以上。偶尔失败的那几次基本都是 Zilow 自己在做 A/B 测试改了页面结构。

## 环境准备与依赖安装

先确保你本地有 Node.js 16+。然后初始化项目：

```bash
mkdir zillow-scraper && cd zillow-scraper
npm init -y
npm install axios cheerio
```

只需要两个依赖：

- `axios`：发 HTTP 请求
- `cheerio`：解析 HTML，语法跟 jQuery 一样

不需要 Puppeteer，不需要 playwright，不需要任何无头浏览器。反爬那层全交给 ScraperAPI。

## 核心代码：抓取 Zillow 搜索结果页

```javascript
const axios = require('axios');
const cheerio = require('cheerio');

const API_KEY = '你的ScraperAPI密钥';
const BASE_URL = 'https://api.scraperapi.com';

async function scrapeZillowListings(zipCode) {
  const targetUrl = `https://www.zillow.com/homes/for_sale/${zipCode}_rb/`;

  try {
    const response = await axios.get(BASE_URL, {
      params: {
        api_key: API_KEY,
        url: targetUrl,
        render: 'true',
        country_code: 'us'
      },
      timeout: 6000
    });

    const $ = cheerio.load(response.data);
    const listings = [];

    // Zilow 把房源数据塞在一个 script 标签的 JSON 里
    const scriptTags = $('script[type="application/json"]');
    scriptTags.each((i, el) => {
      const content = $(el).html();
      if (content && content.includes('listResults')) {
        try {
          const data = JSON.parse(content);
          const results = extractListings(data);
          listings.push(...results);
        } catch (e) {
          // 不是目标 JSON，跳过
        }
      }
    });

    return listings;
  } catch (error) {
    console.error(`抓取失败: ${error.message}`);
    return [];
  }
}

function extractListings(data) {
  const listings = [];
  // Zillow 的数据结构经常变，这里做防御性解析
  const searchResults = findNestedKey(data, 'listResults') 
    || findNestedKey(data, 'searchResults');
  
  if (!searchResults) return listings;

  const items = searchResults.listResults || searchResults || [];
  
  for (const item of items) {
    listings.push({
      address: item.address || '未知地址',
      price: item.price || item.unformattedPrice || '价格未公开',
      beds: item.beds || null,
      baths: item.baths || null,
      area: item.area || null,
      detailUrl: item.detailUrl || null,
      statusText: item.statusText || '在售'
    });
  }

  return listings;
}

// 递归查找嵌套 JSON 中的目标 key
function findNestedKey(obj, targetKey) {
  if (!obj || typeof obj !== 'object') return null;
  if (obj[targetKey]) return obj[targetKey];
  
  for (const key of Object.keys(obj)) {
    const result = findNestedKey(obj[key], targetKey);
    if (result) return result;
  }
  return null;
}
```

这段代码的关键点：`render: 'true'` 参数让 ScraperAPI 用无头浏览器渲染页面，这样 Zillow 通过 JS 动态注入的数据才能被拿到。

## 处理分页：批量抓取多页结果

Zillow 搜索结果通常有多页。分页逻辑很直接：

```javascript
async function scrapeAllPages(zipCode, maxPages = 5) {
  const allListings = [];

  for (let page = 1; page <= maxPages; page++) {
    console.log(`正在抓取第 ${page} 页...`);
    const targetUrl = `https://www.zillow.com/homes/for_sale/${zipCode}_rb/${page}_/`;

    const response = await axios.get(BASE_URL, {
      params: {
        api_key: API_KEY,
        url: targetUrl,
        render: 'true',
        country_code: 'us'
      },
      timeout: 60000
    });

    const $ = cheerio.load(response.data);
    const pageListings = parseListingsFromHtml($);
    if (pageListings.length === 0) {
      console.log(`第 ${page} 页无数据，停止翻页`);
      break;
    }

    allListings.push(...pageListings);
    
    // 每页之间加个随机延迟，别太激进
    const delay = 2000 + Math.random() * 3000;
    await new Promise(resolve => setTimeout(resolve, delay));
  }

  console.log(`共抓取 ${allListings.length} 条房源`);
  return allListings;
}
```

加随机延迟不是因为 ScraperAPI 需要——它自己会处理请求节奏——而是给你的代码一个缓冲，避免并发太高把自己的 API 额度一下子烧完。

## 抓取单个房源详情页

搜索结果页的数据比较粗。如果你需要更详细的信息（房屋年份、HOA 费用、学区评分等），得进详情页：

```javascript
async function scrapeListingDetail(detailUrl) {
  const response = await axios.get(BASE_URL, {
    params: {
      api_key: API_KEY,
      url: detailUrl,
      render: 'true',
      country_code: 'us'
    },
    timeout: 60000
  });

  const $ = cheerio.load(response.data);
  // 详情页的数据同样在 JSON-LD 或内嵌 script 里
  const detail = {};

  // 尝试从 JSON-LD 提取
  const jsonLd = $('script[type="application/ld+json"]').first().html();
  if (jsonLd) {
    try {
      const structured = JSON.parse(jsonLd);
      detail.description = structured.description || '';
      detail.latitude = structured.geo?.latitude || null;
      detail.longitude = structured.geo?.longitude || null;
    } catch (e) {}
  }

  // 从页面元素提取补充信息
  detail.yearBuilt = extractFact($, 'Year built');
  detail.lotSize = extractFact($, 'Lot size');
  detail.hoaFee = extractFact($, 'HOA');
  detail.propertyType = extractFact($, 'Type');

  return detail;
}

function extractFact($, label) {
  const factElements = $('[data-testid="bed-bath-beyond"] span, .dpf__sc-1me8eh6-0');
  let value = null;
  
  factElements.each((i, el) => {
    const text = $(el).text().trim();
    if (text.toLowerCase().includes(label.toLowerCase())) {
      value = text.replace(label, '').replace(':', '').trim();
    }
  });
  
  return value;
}
```

## 数据导出：存成 CSV 方便后续分析

```javascript
const fs = require('fs');

function exportToCSV(listings, filename = 'zillow_data.csv') {
  const headers = ['地址', '价格', '卧室', '浴室', '面积(sqft)', '状态', '详情链接'];
  const rows = listings.map(item => [
    `"${item.address}"`,
    item.price,
    item.beds || '',
    item.baths || '',
    item.area || '',
    item.statusText,
    item.detailUrl || ''
  ]);

  const csv = [headers.join(','), ...rows.map(r => r.join(','))].join('\n');
  fs.writeFileSync(filename, csv, 'utf-8');
  console.log(`数据已导出到 ${filename}`);
}
```

## 完整运行示例

把上面的模块串起来：

```javascript
async function main() {
  const zipCode = '90210'; // 比弗利山庄，测试用
  console.log(`开始抓取 ZIP: ${zipCode} 的 Zillow 房源数据`);
  
  const listings = await scrapeAllPages(zipCode, 3);
  
  if (listings.length > 0) {
    exportToCSV(listings, `zillow_${zipCode}.csv`);
    // 抓前 3 个房源的详情作为演示
    for (let i = 0; i < Math.min(3, listings.length); i++) {
      if (listings[i].detailUrl) {
        const detail = await scrapeListingDetail(listings[i].detailUrl);
        console.log(`${listings[i].address} 详情:`, detail);
        await new Promise(r => setTimeout(r, 2000));
      }
    }
  }
}

main().catch(console.error);
```

## ScraperAPI 套餐怎么选：看你的抓取量决定

我刚开始用的时候选错了套餐，后来升级了一次。这里把所有套餐列出来，你根据自己的量来选：

| 套餐名 | API 请求额度 | 并发数 | 价格 | 适合谁 | 专属链接 |
| ------ | ------------- | ------ | -------- | ---------- | --- |
| Hobby | 100,000 次/月 | 20 | $49/月 | 个人项目、小规模数据采集 | [锁定 Hobby 套餐开始抓取](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 500,000 次/月 | 50 | $149/月 | 中等规模爬虫、多城市房源监控 | [获取 Startup 套餐的完整并发能力](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 次/月 | 100 | $299/月 | 商业级数据产品、全美房源追踪 | [解锁 Business 套餐百万级额度](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义 | 自联系销售 | 大型数据平台、需要定制 SLA | [联系销售获取企业定制方案](https://www.scraperapi.com/?fp_ref=coupons) |   |

所有付费套餐都有 7 天免费试用。注册就送 5000 次免费请求，不绑卡也能用，这点我觉得挺厚道的——你可以先跑通代码确认能用，再决定要不要付费。

👉 [先用免费额度跑通你的 Zilow 抓取流程](https://www.scraperapi.com/?fp_ref=coupons)

## 踩坑记录与优化技巧

### Zillow 页面结构经常变怎么办

这是最头疼的问题。Zilow 大概每隔几周就会调整一次 DOM 结构或者 JSON 数据的嵌套方式。我的应对策略：

1. **优先解析 JSON 而非 DOM**：Zillow 的核心数据都在 `<script>` 标签里的 JSON 中，比 CSS 选择器稳定得多
2. **用递归查找代替固定路径**：上面代码里的 `findNestedKey` 函数就是干这个的，不管 Zillow 怎么改嵌套层级，只要 key 名没变就能找到
3. **加错误兜底**：每个解析步骤都 try-catch，单条数据解析失败不影响整体

### render 参数什么时候该开

`render=true` 会消耗更多 API 额度（大约是普通请求的 5-10 倍）。我的经验是：

- 搜索结果页：**必须开**，数据全靠 JS 渲染
- 详情页：**也建议开**，虽然部分数据在初始 HTML 里，但完整信息需要渲染
- 如果你只需要非常基础的信息（比如只要价格和地址），可以试不开 render，看返回的 HTML 里有没有你要的数据

### 并发控制

别一次性发几十个请求。即使 ScraperAPI 能扛住，你的代码逻辑也容易出问题。我一般控制在 3-5 个并发：

```javascript
async function batchScrape(urls, concurrency = 3) {
  const results = [];
  for (let i = 0; i < urls.length; i += concurrency) {
    const batch = urls.slice(i, i + concurrency);
    const batchResults = await Promise.all(
      batch.map(url => scrapeZillowListings(url).catch(() => []))
    );
    results.push(...batchResults.flat());
    if (i + concurrency < urls.length) {
      await new Promise(r => setTimeout(r, 3000));
    }
  }
  
  return results;
}
```

## 常见问题

### ScraperAPI 返回的 HTML 是空的或者很短怎么办？

大概率是 render 参数没开，或者超时时间设太短了。Zillow 页面渲染需要时间，我建议 timeout 至少设 60 秒。另外检查一下你的 API key 是不是过期了。

### 抓取频率多高比较安全？

用 ScraperAPI 的话，频率限制主要看你的套餐并发数。Hobby 套餐 20 并发，意味着你同时最多跑 20 个请求。我个人习惯每个请求之间加 2-5 秒随机延迟，一天跑个几千条完全没问题。

### 能抓到已售房源（sold listings）吗？

能。把 URL 里的 `for_sale` 改成 `recently_sold` 就行：

```javascript
const targetUrl = `https://www.zillow.com/homes/recently_sold/${zipCode}_rb/`;
```

解析逻辑基本一样，只是返回的字段会多一个成交价和成交日期。

### 抓下来的数据能商用吗？

这个问题不在技术层面。Zillow 的 Terms of Service 对数据使用有限制，你需要自己评估合规性。ScraperAPI 只是帮你获取公开可访问的网页内容，怎么用是你的事。

### 为什么不直接用 Zilow 的官方 API？

Zillow 在2021 年关闭了公开的房产数据 API（原来的 Zestimate API）。现在他们的 Bridge API 只对合作伙伴开放，申请门槛很高。对大多数开发者来说，网页抓取是唯一现实的选择。

### ScraperAPI 和自建代理池比哪个划算？

我两种都用过。自建代理池前期成本低，但维护成本高——你得自己处理代理失效、IP 被封、浏览器指纹更新这些事。如果你的抓取量不是特别大（月请求量在百万以下），用 ScraperAPI 省下来的开发和维护时间远比套餐费值钱。ScraperAPI 运营超过 5 年了，代理池覆盖全球，稳定性这块我没什么可抱怨的。

---

## 最后说两句

Zillow 抓取这件事，技术难度其实不在解析——cheerio 配合 JSON 递归查找就够了。难的是怎么稳定地拿到渲染后的完整 HTML。自己搞反爬对抗是个无底洞，Zillow 那边有专门的团队在升级防护，你一个人耗不过他们。

把反爬这层外包出去，自己专注写业务逻辑和数据分析，这是我折腾了半年之后得出的结论。代码我都贴在上面了，复制过去改个 ZIP code 就能跑。

👉 [注册 ScraperAPI 拿 5000 次免费请求，直接跑通你的 Zillow 抓取代码](https://www.scraperapi.com/?fp_ref=coupons)

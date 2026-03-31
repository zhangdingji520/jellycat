<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Jellycat Nesting Bunnies 搜索脚本</title>
    <style>
        body {
            background: #1e1e2f;
            color: #e0e0e0;
            font-family: 'SF Mono', 'Menlo', 'Cascadia Code', monospace;
            line-height: 1.6;
            padding: 2rem;
            max-width: 1200px;
            margin: 0 auto;
        }
        h1, h2 {
            color: #ffcc66;
            font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
        }
        h1 {
            border-bottom: 2px solid #ffcc66;
            padding-bottom: 0.5rem;
        }
        pre {
            background: #2d2d3a;
            padding: 1.5rem;
            border-radius: 12px;
            overflow-x: auto;
            font-size: 0.85rem;
            border: 1px solid #3a3a4a;
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
        }
        code {
            font-family: inherit;
        }
        .button-group {
            margin: 1rem 0 2rem;
            display: flex;
            gap: 1rem;
            flex-wrap: wrap;
        }
        button {
            background: #3a3a4a;
            border: none;
            color: #ffcc66;
            padding: 0.6rem 1.2rem;
            font-size: 1rem;
            font-weight: bold;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.2s ease;
            font-family: system-ui;
        }
        button:hover {
            background: #ffcc66;
            color: #1e1e2f;
        }
        .note {
            background: #2a2a36;
            padding: 1rem;
            border-left: 4px solid #ffcc66;
            margin: 1.5rem 0;
            font-family: system-ui;
        }
        footer {
            margin-top: 3rem;
            text-align: center;
            font-size: 0.8rem;
            color: #888;
            border-top: 1px solid #3a3a4a;
            padding-top: 1.5rem;
        }
    </style>
</head>
<body>
    <h1>🐰 Jellycat Nesting Bunnies 搜索脚本</h1>
    <p>使用 Serper.dev API 搜索 Jellycat Nesting Bunnies 系列，收集独立域名，验证库存，排除美元网站，生成 HTML 报告。</p>
    <div class="button-group">
        <button id="copyBtn">📋 复制代码</button>
        <button id="downloadBtn">💾 下载脚本 (.py)</button>
    </div>
    <div class="note">
        ⚠️ 注意：运行前请确保已安装 <code>requests</code> 库（Pythonista 已内置），并将脚本中的 <code>SERPER_API_KEY</code> 替换为您自己的密钥。
    </div>
    <pre id="codeBlock"><code># jellycat_nesting_bunnies.py
# 搜索 Jellycat Nesting Bunnies 系列玩偶，收集独立域名，验证库存，排除美元网站，生成 HTML 报告
# 使用 Serper.dev API

import requests
import json
import time
from urllib.parse import urlparse
from concurrent.futures import ThreadPoolExecutor, as_completed

# ================== 配置 ==================
SERPER_API_KEY = "17e141da4fb66b674c2a5b046afb598834bf28f2"  # 请替换为您的密钥

# 目标国家及本地化搜索词（每个国家一组）
COUNTRIES_CONFIG = [
    {"code": "DE", "name": "Germany", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat nestkaninchen", 
        "jellycat schachtelhasen"
    ]},
    {"code": "PL", "name": "Poland", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat króliki gniazdujące"
    ]},
    {"code": "NL", "name": "Netherlands", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat nestkonijntjes"
    ]},
    {"code": "BE", "name": "Belgium", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat lapins gigognes",
        "jellycat nestkonijntjes"
    ]},
    {"code": "PT", "name": "Portugal", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat coelhos aninhados"
    ]},
    {"code": "ES", "name": "Spain", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat conejos anidados"
    ]},
    {"code": "FR", "name": "France", "local_keywords": [
        "jellycat nesting bunnies", 
        "jellycat lapins gigognes"
    ]}
]

COMMON_KEYWORDS = [
    "jellycat nesting bunnies",
    "nesting bunnies jellycat",
    "jellycat nesting bunnies set",
    "jellycat nesting rabbits",
    "jellycat stacking bunnies"
]

EXCLUDE_DOMAINS = [
    "jellycat.com", "eu.jellycat.com", "de.jellycat.com", "fr.jellycat.com",
    "amazon", "ebay", "walmart", "target", "toysrus", "wish", "etsy", "zalando",
    "bol.com", "aliexpress", "cdiscount", "fnac", "darty", "otto",
    "vinted", "depop", "allegro", "marktplaats", "olx", "wallapop", "leboncoin",
    "hood.de", "kleinanzeigen",
    "facebook", "instagram", "tiktok", "twitter", "pinterest", "youtube",
    "jellyjournal.com", "lilietmilou.com", "ubuy"
]

SOLD_OUT_KEYWORDS = [
    "sold out", "out of stock", "no stock", "not available", "currently unavailable",
    "ausverkauft", "nicht vorrätig", "nicht auf lager", "momentan nicht verfügbar",
    "derzeit nicht vorrätig", "derzeit nicht verfügbar",
    "nicht lieferbar", "vergriffen", "leider ausverkauft",
    "artikel nicht verfügbar", "nicht mehr vorrätig", "nicht auf Lager", "Nicht vorrätig",
    "niedostępny", "brak w magazynie", "wyprzedane", "chwilowo niedostępny",
    "uitverkocht", "niet op voorraad", "niet beschikbaar", "tijdelijk niet beschikbaar",
    "niet leverbaar", "momenteel niet op voorraad", "op=op", "niet meer leverbaar",
    "épuisé", "rupture de stock", "plus disponible", "indisponible",
    "temporairement indisponible", "en rupture", "produit indisponible",
    "non disponible", "hors stock",
    "esgotado", "fora de stock", "indisponível", "não disponível",
    "sem stock", "temporariamente indisponível", "produto esgotado",
    "agotado", "sin stock", "no disponible", "fuera de stock",
    "temporalmente no disponible", "artículo agotado", "no hay stock"
]

IN_STOCK_KEYWORDS = [
    "in stock", "available", "in store", "ready to ship",
    "auf lager", "sofort lieferbar", "lieferbar", "verfügbar", "auf Lager",
    "na stanie", "dostępny", "w magazynie", "dostępna",
    "op voorraad", "leverbaar", "beschikbaar",
    "en stock", "disponible",
    "em stock", "disponível", "em estoque",
    "en stock", "disponible", "en existencias"
]

USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"

# ================== 搜索函数 ==================
def search_serper(query, country_code):
    url = "https://google.serper.dev/search"
    payload = json.dumps({"q": query, "gl": country_code})
    headers = {
        'X-API-KEY': SERPER_API_KEY,
        'Content-Type': 'application/json'
    }
    try:
        resp = requests.post(url, headers=headers, data=payload, timeout=15)
        if resp.status_code != 200:
            return []
        data = resp.json()
        organic = data.get("organic", [])
        return [item.get("link") for item in organic if item.get("link")]
    except Exception:
        return []

def is_valid_url(url):
    if not url:
        return False
    domain = urlparse(url).netloc.replace("www.", "")
    return not any(ex in domain.lower() for ex in EXCLUDE_DOMAINS)

def collect_urls(target=200):
    collected = {}
    total_requests = 0
    print(f"=== 搜索 Jellycat Nesting Bunnies，目标 {target} 个独立域名 ===\n")
    for cfg in COUNTRIES_CONFIG:
        if len(collected) >= target:
            break
        print(f"🇪🇺 国家: {cfg['name']} ({cfg['code']})")
        all_keywords = cfg["local_keywords"] + COMMON_KEYWORDS
        for kw in all_keywords:
            if len(collected) >= target:
                break
            print(f"  搜索: {kw}")
            urls = search_serper(kw, cfg["code"])
            total_requests += 1
            if not urls:
                continue
            for url in urls:
                if len(collected) >= target:
                    break
                if is_valid_url(url):
                    domain = urlparse(url).netloc.replace("www.", "")
                    if domain not in collected:
                        collected[domain] = url
                        print(f"      已收集 {len(collected)} / {target}  {url[:80]}")
            time.sleep(1.1)
    print(f"\n收集完成: {len(collected)} 个独立域名，请求次数 {total_requests}\n")
    return list(collected.values())

def check_product_page(url):
    headers = {"User-Agent": USER_AGENT}
    try:
        resp = requests.get(url, headers=headers, timeout=5, allow_redirects=True)
        if resp.status_code != 200:
            return url, False, f"HTTP {resp.status_code}"
        text = resp.text.lower()
        if any(kw in text for kw in SOLD_OUT_KEYWORDS):
            return url, False, None
        if any(kw in text for kw in IN_STOCK_KEYWORDS):
            if ("$" in text or "usd" in text) and not ("€" in text or "eur" in text):
                return url, False, "USD website (排除)"
            return url, True, None
        return url, False, None
    except Exception as e:
        return url, False, str(e)

def verify_urls(urls, workers=30):
    buyable = []
    sold_out = []
    unknown = []
    total = len(urls)
    print(f"=== 多线程验证 {total} 个域名（{workers} 线程）===\n")
    with ThreadPoolExecutor(max_workers=workers) as executor:
        futures = {executor.submit(check_product_page, url): url for url in urls}
        for i, future in enumerate(as_completed(futures), 1):
            url, can_buy, err = future.result()
            if err:
                unknown.append((url, err))
                print(f"[{i}/{total}] ❌ 错误: {url[:70]} - {err}")
            elif can_buy:
                buyable.append(url)
                print(f"[{i}/{total}] ✅ 可购买: {url[:70]}")
            else:
                sold_out.append(url)
                print(f"[{i}/{total}] 🚫 售罄或不可购买: {url[:70]}")
    return buyable, sold_out, unknown

def save_html(filename, items, title):
    with open(filename, "w", encoding="utf-8") as f:
        f.write(f"<html><head><meta charset='UTF-8'><title>{title}</title></head><body>")
        f.write(f"<h1>{title}</h1><p>共 {len(items)} 条</p><ol>")
        if items and isinstance(items[0], tuple):
            for url, reason in items:
                f.write(f'<li><a href="{url}" target="_blank">{url}</a> <span style="color:gray;">#{reason}</span></li>')
        else:
            for url in items:
                f.write(f'<li><a href="{url}" target="_blank">{url}</a></li>')
        f.write("</ol></body></html>")
    print(f"已生成 {filename}")

def main():
    target = 200
    urls = collect_urls(target)
    if not urls:
        print("未收集到任何 URL，请检查网络或 API 密钥。")
        return
    buyable, sold_out, unknown = verify_urls(urls)
    save_html("nesting_bunnies_buyable.html", buyable, "✅ 可购买 Jellycat Nesting Bunnies (无美元网站)")
    save_html("nesting_bunnies_sold_out.html", sold_out, "🚫 售罄或不可购买")
    save_html("nesting_bunnies_unknown.html", unknown, "❓ 无法判断")
    save_html("nesting_bunnies_all_urls.html", urls, "所有抓取到的 Jellycat Nesting Bunnies 商品页")
    print(f"\n🎉 完成！可购买: {len(buyable)}，售罄: {len(sold_out)}，未知: {len(unknown)}")

if __name__ == "__main__":
    main()
</code></pre>

    <footer>
        📦 脚本生成于 2026-04-01 | 使用 Serper.dev API | 适用于 Pythonista 3
    </footer>

    <script>
        const codeBlock = document.getElementById('codeBlock');
        const copyBtn = document.getElementById('copyBtn');
        const downloadBtn = document.getElementById('downloadBtn');

        copyBtn.addEventListener('click', async () => {
            const code = codeBlock.innerText;
            try {
                await navigator.clipboard.writeText(code);
                copyBtn.textContent = '✓ 已复制！';
                setTimeout(() => copyBtn.textContent = '📋 复制代码', 2000);
            } catch (err) {
                alert('复制失败，请手动复制。');
            }
        });

        downloadBtn.addEventListener('click', () => {
            const code = codeBlock.innerText;
            const blob = new Blob([code], { type: 'text/x-python' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'jellycat_nesting_bunnies.py';
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        });
    </script>
</body>
</html>

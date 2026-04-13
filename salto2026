import requests
from bs4 import BeautifulSoup
import json
from datetime import datetime, timedelta
from concurrent.futures import ThreadPoolExecutor
import re


def get_deep_data(event_url):
    h = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0 Safari/537.36'}
    infopacks = []
    form_url = None
    try:
        res = requests.get(event_url, headers=h, timeout=10)
        soup = BeautifulSoup(res.content, 'html.parser')
        downloads_div = soup.select_one('.downloads') or soup.select_one('.downloads-list')
        if downloads_div:
            for a in downloads_div.select('a[href]'):
                href = a.get('href')
                if href and not href.startswith('#'):
                    infopacks.append("https://www.salto-youth.net" + href if href.startswith('/') else href)
        proc_tag = soup.find('a', href=lambda h: h and 'application-procedure' in h)
        if proc_tag:
            p_url = "https://www.salto-youth.net" + proc_tag['href'] if proc_tag['href'].startswith('/') else proc_tag['href']
            res_proc = requests.get(p_url, headers=h, timeout=10)
            soup_proc = BeautifulSoup(res_proc.content, 'html.parser')
            btn = soup_proc.find('a', class_='large-button-inline') or soup_proc.find('a', string=lambda t: t and "Proceed to" in t)
            if btn:
                form_url = btn['href']
    except:
        pass
    return infopacks, form_url


def scrape_salto_complete():
    target_date = datetime.now() + timedelta(days=30)
    d, m, y = str(target_date.day), str(target_date.month), str(target_date.year)
    h = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0 Safari/537.36'}
    all_results = []
    offset = 0
    limit = 10

    with requests.Session() as session:
        while True:
            base_url = "https://www.salto-youth.net/tools/european-training-calendar/browse/"
            params = {
                'b_offset': offset,
                'b_limit': limit,
                'b_order': 'applicationDeadline',
                'b_application_deadline_after_day': d,
                'b_application_deadline_after_month': m,
                'b_application_deadline_after_year': y
            }
            try:
                response = session.get(base_url, params=params, headers=h, timeout=15)
                soup = BeautifulSoup(response.content, 'html.parser')
                items = soup.find_all('div', class_='tool-item')
                if not items:
                    break
                page_tasks = []
                for item in items:
                    name_tag = item.select_one('.tool-item-name a')
                    if not name_tag: continue
                    event_url = "https://www.salto-youth.net" + name_tag['href'] if name_tag['href'].startswith('/') else name_tag['href']
                    event_id = None
                    match = re.search(r'\.(\d+)/?$', event_url)
                    if match: event_id = match.group(1)
                    desc_div = item.select_one('.tool-item-description')
                    summary = None
                    if desc_div:
                        for p in desc_div.find_all('p'):
                            text = p.get_text(strip=True)
                            if text and len(text) > 50:
                                summary = text
                                break
                    page_tasks.append({
                        "event_id": event_id,
                        "category": item.select_one('.tool-item-category').get_text(strip=True) if item.select_one('.tool-item-category') else None,
                        "title": name_tag.get_text(strip=True),
                        "url": event_url,
                        "summary": summary,
                        "participants_from": item.select_one('p.tightened-bodycopy').get_text(strip=True) if item.select_one('p.tightened-bodycopy') else None,
                        "date": desc_div.find('p', class_='h5').get_text(strip=True) if desc_div and desc_div.find('p', class_='h5') else None,
                        "location": desc_div.find('p', class_='microcopy').get_text(strip=True) if desc_div and desc_div.find('p', class_='microcopy') else None,
                        "deadline": item.select_one('.callout-module .h3').get_text(strip=True) if item.select_one('.callout-module .h3') else None,
                    })
                with ThreadPoolExecutor(max_workers=5) as executor:
                    futures = {executor.submit(get_deep_data, p["url"]): p for p in page_tasks}
                    for future in futures:
                        p = futures[future]
                        infopacks, form = future.result()
                        p["infopack"] = ", ".join(infopacks) if infopacks else None
                        p["application_form"] = form
                        all_results.append(p)
                offset += limit
                if len(items) < limit or offset >= 100:
                    break
            except:
                break
    return all_results


# ESECUZIONE
output = scrape_salto_complete()

# Salva il JSON nel file
with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(output, f, indent=4, ensure_ascii=False)

print(f"Salvati {len(output)} eventi")

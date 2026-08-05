#!/usr/bin/env python3
import json, math, re, statistics, sys
from datetime import datetime, timezone
from pathlib import Path
from urllib.parse import urljoin

import requests
from bs4 import BeautifulSoup

RACE_URL = 'https://results.zone/scm-24hour-2026/races/8907/results'
OUT = Path(__file__).with_name('data.json')
TARGETS = [
    {'key':'Иван Заборский','site':'Заборский Иван','color':'#2457F5'},
    {'key':'Сергей Вербицкий','site':'Вербицкий Сергей','color':'#FF6B0B'},
    {'key':'Екатерина Азарова','site':'Азарова Екатерина','color':'#875AF5'},
    {'key':'Андрей Толстопятенко','site':'Толстопятенко Андрей','color':'#079669'},
    {'key':'Евгений Однорог','site':'Однорог Евгений','color':'#DF3760'},
]
UA='Mozilla/5.0 (compatible; ultra-dashboard-updater/1.0; +https://github.com/)'

def norm(s):
    return re.sub(r'\s+',' ',s.replace('\xa0',' ')).strip().casefold()

def parse_duration(s):
    s=s.strip().replace(',', '.')
    parts=s.split(':')
    try:
        if len(parts)==2:
            m,sec=parts; return float(m)*60+float(sec)
        if len(parts)==3:
            h,m,sec=parts; return float(h)*3600+float(m)*60+float(sec)
    except ValueError:
        pass
    raise ValueError(f'bad duration: {s!r}')

def discover_urls(session):
    found={}
    for page in range(1,6):
        r=session.get(RACE_URL,params={'page':page},timeout=30)
        r.raise_for_status()
        soup=BeautifulSoup(r.text,'html.parser')
        links=soup.find_all('a',href=re.compile(r'/scm-24hour-2026/races/8907/results/\d+$'))
        if not links and page>1: break
        for a in links:
            txt=norm(a.get_text(' ',strip=True))
            href=urljoin(RACE_URL,a.get('href'))
            for t in TARGETS:
                if norm(t['site']) in txt:
                    found[t['key']]=href
        if len(found)==len(TARGETS): break
    missing=[t['site'] for t in TARGETS if t['key'] not in found]
    if missing:
        raise RuntimeError('Не найдены страницы: '+', '.join(missing))
    return found

def local_baselines(segs):
    out=[]
    n=len(segs)
    for i in range(n):
        w=segs[max(0,i-12):min(n,i+13)]
        plausible=[x for x in w if 180<=x<=1200]
        if not plausible: plausible=w[:]
        med=statistics.median(plausible)
        tighter=[x for x in plausible if x<=med*1.45]
        out.append(statistics.median(tighter or plausible))
    return out

def parse_athlete(html, expected_name, color):
    soup=BeautifulSoup(html,'html.parser')
    h1=soup.find('h1')
    if h1 and norm(expected_name) not in norm(h1.get_text(' ',strip=True)):
        raise RuntimeError(f'Открылась не та страница: {h1.get_text(" ",strip=True)}')
    table=soup.select_one('#heat_splits_table')
    if table is None: raise RuntimeError(f'Нет таблицы сплитов у {expected_name}')
    raw=[]
    for tr in table.find_all('tr'):
        cells=tr.find_all(['th','td'])
        if len(cells)<7: continue
        split=cells[0].get_text(' ',strip=True)
        m=re.match(r'Круг\s+(\d+)',split)
        if not m: continue
        lap=int(m.group(1))
        elapsed=parse_duration(cells[3].get_text(' ',strip=True))
        seg=parse_duration(cells[5].get_text(' ',strip=True))
        raw.append((lap,elapsed,seg))
    if not raw: raise RuntimeError(f'Нет километровых кругов у {expected_name}')
    segs=[x[2] for x in raw]
    bases=local_baselines(segs)
    laps=[]
    for (lap,elapsed,seg),base in zip(raw,bases):
        is_rest=(seg-base>=180 and seg>=base*1.30)
        rest=max(0.0,seg-base) if is_rest else 0.0
        move=base if is_rest else seg
        hour=(12+elapsed/3600)%24
        laps.append({
            'lap':lap,'elapsed':round(elapsed,2),'seg':round(seg,2),'km':lap,
            'base':round(base,1),'rest':round(rest,1),'move':round(move,1),
            'day':int(elapsed//86400)+1,'hour':hour,
        })
    return {'color':color,'laps':laps}

def main():
    s=requests.Session();s.headers.update({'User-Agent':UA,'Accept-Language':'ru,en;q=0.8'})
    urls=discover_urls(s)
    data={}
    for t in TARGETS:
        u=urls[t['key']]
        r=s.get(u,timeout=30);r.raise_for_status()
        data[t['key']]=parse_athlete(r.text,t['site'],t['color'])
        print(f"{t['key']}: {len(data[t['key']]['laps'])} км · {u}")
    payload={
        'updated_at':datetime.now(timezone.utc).isoformat().replace('+00:00','Z'),
        'source':RACE_URL,
        'athlete_urls':urls,
        'data':data,
    }
    tmp=OUT.with_suffix('.json.tmp')
    tmp.write_text(json.dumps(payload,ensure_ascii=False,separators=(',',':'))+'\n',encoding='utf-8')
    tmp.replace(OUT)
    print('written',OUT)

if __name__=='__main__':
    main()

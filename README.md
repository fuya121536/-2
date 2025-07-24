# -2
// scraping.js
import puppeteer from 'puppeteer-extra';
import StealthPlugin from 'puppeteer-extra-plugin-stealth';
import { createClient } from '@supabase/supabase-js';

puppeteer.use(StealthPlugin());

const proxy = process.env.PROXY_SERVER || '';
const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

const MACHINE_URLS = [
  { name: 'アイムTP', url: 'https://...TP&list_kind=day' },
  { name: 'マイV',   url: 'https://...VKD&list_kind=day' },
  { name: 'ハッピーV3', url: 'https://...EA&list_kind=day' },
  { name: 'ゴーゴー3',  url: 'https://...3KA&list_kind=day' },
];

(async () => {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      proxy && `--proxy-server=${proxy}`,
    ].filter(Boolean),
  });
  const page = await browser.newPage();
  await page.setUserAgent(
    'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148'
  );

  const today = new Date().toISOString().slice(0, 10); // YYYY-MM-DD

  for (const m of MACHINE_URLS) {
    await page.goto(m.url, { waitUntil: 'networkidle2', timeout: 0 });

    // ページ内のテーブルを取得
    const rows = await page.$$eval('table.tbl_data01 tr', trs =>
      trs.slice(1).map(tr => {
        const tds = [...tr.querySelectorAll('td')].map(td => td.innerText.trim());
        return {
          number: parseInt(tds[0], 10),
          big:    parseInt(tds[1], 10),
          reg:    parseInt(tds[2], 10),
          spins:  parseInt(tds[3].replace(/,/g, ''), 10),
        };
      })
    );

    // 判別 & Supabase へ保存
    for (const r of rows) {
      const totalBonus = r.big + r.reg;
      const combined   = totalBonus ? (r.spins / totalBonus) : null;
      const regProb    = r.reg     ? (r.spins / r.reg)      : null;

      let expect6 = 0;
      if (regProb && regProb < 270 && r.spins > 2000) expect6 = 0.8;
      else if (regProb && regProb < 300)             expect6 = 0.5;

      await supabase.from('juggler_history').insert({
        date: today,
        machine: m.name,
        number: r.number,
        big: r.big,
        reg: r.reg,
        spins: r.spins,
        combined,
        reg_prob: regProb,
        expect6,
      });
    }
    console.log(`${m.name} done`);
  }

  await browser.close();
})();
// scraping.js
import puppeteer from 'puppeteer-extra';
import StealthPlugin from 'puppeteer-extra-plugin-stealth';
import { createClient } from '@supabase/supabase-js';

puppeteer.use(StealthPlugin());

const proxy = process.env.PROXY_SERVER || '';
const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

const MACHINE_URLS = [
  { name: 'アイムTP', url: 'https://...TP&list_kind=day' },
  { name: 'マイV',   url: 'https://...VKD&list_kind=day' },
  { name: 'ハッピーV3', url: 'https://...EA&list_kind=day' },
  { name: 'ゴーゴー3',  url: 'https://...3KA&list_kind=day' },
];

(async () => {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      proxy && `--proxy-server=${proxy}`,
    ].filter(Boolean),
  });
  const page = await browser.newPage();
  await page.setUserAgent(
    'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148'
  );

  const today = new Date().toISOString().slice(0, 10); // YYYY-MM-DD

  for (const m of MACHINE_URLS) {
    await page.goto(m.url, { waitUntil: 'networkidle2', timeout: 0 });

    // ページ内のテーブルを取得
    const rows = await page.$$eval('table.tbl_data01 tr', trs =>
      trs.slice(1).map(tr => {
        const tds = [...tr.querySelectorAll('td')].map(td => td.innerText.trim());
        return {
          number: parseInt(tds[0], 10),
          big:    parseInt(tds[1], 10),
          reg:    parseInt(tds[2], 10),
          spins:  parseInt(tds[3].replace(/,/g, ''), 10),
        };
      })
    );

    // 判別 & Supabase へ保存
    for (const r of rows) {
      const totalBonus = r.big + r.reg;
      const combined   = totalBonus ? (r.spins / totalBonus) : null;
      const regProb    = r.reg     ? (r.spins / r.reg)      : null;

      let expect6 = 0;
      if (regProb && regProb < 270 && r.spins > 2000) expect6 = 0.8;
      else if (regProb && regProb < 300)             expect6 = 0.5;

      await supabase.from('juggler_history').insert({
        date: today,
        machine: m.name,
        number: r.number,
        big: r.big,
        reg: r.reg,
        spins: r.spins,
        combined,
        reg_prob: regProb,
        expect6,
      });
    }
    console.log(`${m.name} done`);
  }

  await browser.close();
})();
name: Daily Scrape

on:
  # 日本時間 06:00 = UTC 21:00
  schedule:
    - cron: '0 21 * * *'
  workflow_dispatch:   # 手動トリガ

jobs:
  scrape:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - name: Install deps
        run: |
          corepack enable
          pnpm i --frozen-lockfile

      - name: Run scraper
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
          PROXY_SERVER: ${{ secrets.PROXY_SERVER }}
        run: pnpm scrape

      

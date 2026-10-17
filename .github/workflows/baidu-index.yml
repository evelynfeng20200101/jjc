// ============================================================
// 百度指数每日采集：由 GitHub Actions 定时运行，产出 baidu-index.json
// 看板的 index.html 读取同目录下的这个文件。
//
// 为什么非要开一个真浏览器：
//   百度指数的数据接口要求请求头里带一个签名（Cipher-Text），
//   这个签名只能由百度页面里的风控脚本在浏览器中现场算出来。
//   所以我们让无头浏览器打开百度指数页面，借它的脚本算签名，再取数据。
//
// 拿到的数据是加密的（简单的字母替换），还要再请求一次密钥来解密。
//
// 环境变量：
//   BDUSS        百度登录凭证，只放在 GitHub Secrets 里，绝不写进代码
//   BAIDU_WORD   关键词，默认「嫁金钗」
//   BAIDU_DAYS   取最近多少天，默认 90
//   BAIDU_PROBE  设为 1 时进入「探测模式」：不需要账号，只测 GitHub 机房 IP 会不会被百度拦
// ============================================================
import { chromium, devices } from 'playwright';
import fs from 'node:fs';

const WORD  = process.env.BAIDU_WORD || '嫁金钗';
const DAYS  = Number(process.env.BAIDU_DAYS || 90);
const OUT   = 'baidu-index.json';
const BDUSS = process.env.BDUSS;
const PROBE = process.env.BAIDU_PROBE === '1';

if (!BDUSS && !PROBE) {
  console.error('::error::没有找到 BDUSS。请先在仓库 Settings → Secrets 里添加，或者先用探测模式运行');
  process.exit(2);
}

// 百度按北京时间出数，日期都按北京时间算
const bjDate = (offsetDays = 0) =>
  new Date(Date.now() + offsetDays * 864e5 + 8 * 3600e3).toISOString().slice(0, 10);
const compact = d => d.replace(/-/g, '');

// 百度每天中午到下午才出前一天的数，所以 workflow 会在白天跑好几次。
// 如果文件里已经是昨天的数，就直接结束：省资源，也避免重复提交。
if (!PROBE && fs.existsSync(OUT)) {
  try {
    const old = JSON.parse(fs.readFileSync(OUT, 'utf8'));
    if (old.asof === bjDate(-1)) {
      console.log(`数据已是最新（${old.asof}），本次跳过`);
      process.exit(0);
    }
  } catch { /* 文件损坏就重新抓 */ }
}

// 解密：密钥前一半是密文字符，后一半是对应的明文字符
function decode(key, cipher) {
  const half = key.length / 2, table = {};
  for (let i = 0; i < half; i++) table[key[i]] = key[half + i];
  return [...cipher].map(c => table[c] ?? ',').join('');
}

const browser = await chromium.launch({ args: ['--no-sandbox'] });
try {
  const context = await browser.newContext({ ...devices['iPhone 13'] });
  if (BDUSS) await context.addCookies([{ name: 'BDUSS', value: BDUSS, domain: '.baidu.com', path: '/' }]);
  const page = await context.newPage();

  // 备用方案：记录页面自己发出的数据请求，主方案失败时拿来用
  const seen = [];
  page.on('response', async r => {
    if (/getIndex|ptbk/.test(r.url())) { try { seen.push({ url: r.url(), body: await r.text() }); } catch {} }
  });

  await page.goto('https://index.baidu.com/baidu-index-mobile/?from=pinzhuan#/compare/' + encodeURIComponent(WORD),
                  { waitUntil: 'domcontentloaded', timeout: 90000 });

  let result;
  try {
    // 主方案：等页面的风控脚本加载好，借它算签名，再自己发请求
    await page.waitForFunction(() => window.Paris && typeof window.Paris === 'object', { timeout: 20000 });
    result = await page.evaluate(async ({ word, start, end }) => {
      const sign = await new Promise(done => {
        if (typeof Paris.getAcsInstance !== 'function') return done('');
        Paris.getAcsInstance((err, inst) => {
          if (err || !inst) return done('');
          inst.getSign((e, s) => done(e ? '' : s));
        });
      });
      const headers = { 'X-Requested-With': 'XMLHttpRequest', 'Cipher-Text': sign };
      const idx = await (await fetch(`/Interface/Newwordgraph/getIndex?region=0&startdate=${start}&enddate=${end}` +
                                     `&wordlist%5B0%5D=${encodeURIComponent(word)}`, { headers })).json();
      if (idx.status !== 0) return { error: `getIndex 返回 ${idx.status} ${idx.message || ''}`, status: idx.status };
      const key = await (await fetch('/Interface/api/ptbk?uniqid=' + idx.uniqid, { headers })).json();
      if (key.status !== 0) return { error: `ptbk 返回 ${key.status}` };
      return { node: idx.data[0].index[0], key: key.data };
    }, { word: WORD, start: compact(bjDate(1 - DAYS)), end: compact(bjDate(0)) });
  } catch (e) {
    console.warn('主方案失败（' + e.message + '），改用页面自身的请求');
    await page.getByText('近90天').first().click({ timeout: 10000 }).catch(() => {});
    await page.waitForTimeout(6000);
    const idx = seen.filter(x => x.url.includes('getIndex')).pop();
    const key = seen.filter(x => x.url.includes('ptbk')).pop();
    if (!idx || !key) throw new Error('页面没有发出数据请求，两种方案都失败');
    const ij = JSON.parse(idx.body), kj = JSON.parse(key.body);
    result = ij.status === 0 ? { node: ij.data[0].index[0], key: kj.data } : { error: 'getIndex 返回 ' + ij.status, status: ij.status };
  }

  if (PROBE) {
    // 没登录时，百度正常应该回「未登录」(10000)；回 10018 说明机房 IP 被风控
    if (result.status === 10000) { console.log('探测通过：签名被接受，GitHub 机房 IP 没被拦。可以去配 BDUSS 了'); process.exit(0); }
    console.error(`::error::探测失败：${result.error || '未知结果'}（10018 表示机房 IP 被百度风控）`);
    process.exit(3);
  }
  if (result.error) throw new Error(result.error);

  const [startStr] = result.node.period.split('|');
  const values = decode(result.key, result.node._all).split(',').map(Number);
  const t0 = Date.UTC(+startStr.slice(0, 4), +startStr.slice(4, 6) - 1, +startStr.slice(6, 8));
  const days = values.map((v, i) => ({ date: new Date(t0 + i * 864e5).toISOString().slice(0, 10), v }))
                     .filter(d => d.v > 0);
  if (!days.length) throw new Error('取到的全是 0：BDUSS 可能过期了，或者被百度风控');

  const out = {
    word: WORD, region: '全国',
    asof: days[days.length - 1].date,
    fetchedAt: new Date().toISOString(),
    rows: days.map(d => [d.date.slice(5), d.v])
  };
  fs.writeFileSync(OUT, JSON.stringify(out));
  console.log(`成功：截至 ${out.asof}，共 ${out.rows.length} 天，最新 ${out.rows.at(-1)[1]}`);
} finally {
  await browser.close();
}

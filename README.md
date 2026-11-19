# AdvertSpy — Personal Shopify Product & Store Research Dashboard

ملخص بالدارجة: هاد المشروع كيدير scraping على الصفحة العمومية `/products.json`
ديال أي متجر Shopify تزيد الدومين ديالو، كيحفظ المنتجات وتاريخ الأثمنة، وكيحسب
"trend score" بسيط باش يبين شنو المنتجات اللي متكررة فبزاف ديال المتاجر أو فيها
تخفيض أو زادت مؤخرا — وهادشي عوض تخمين المبيعات الحقيقية لي ماشي متاح بلا API
مدفوع (بحال ppspy).

## What this actually does

- You add Shopify store domains to track (e.g. `examplestore.myshopify.com` or
  a custom domain that runs Shopify).
- It fetches each store's public `/products.json` endpoint — enabled by
  default on Shopify stores — and stores every product: title, price,
  compare-at price, image, vendor, type, handle.
- Each scan is saved as a price snapshot, so you get real price history per
  product.
- Products with the same (normalized) title across multiple tracked stores
  are grouped together — a simple, honest proxy for "this is being sold by
  many dropshippers right now," which is the same signal tools like this
  ultimately rely on when they don't have access to a store's real order
  data.
- Dashboard tabs: **Trending** (store count + discount + recency), **Newest**,
  **On sale**.

### What it does NOT do (and why)

Real order counts, ad-spend tracking (Meta/TikTok Ads Library), and website
traffic estimates (SimilarWeb-style) all require either scraping data that
isn't public, or a paid third-party API. This project doesn't fake those
numbers. If you want one of those later, the ad tracker and traffic modules
can be added as separate features once you have API access for them.

## Stack

Next.js 14 (App Router, TypeScript) + Prisma + Postgres + Tailwind. No
external account is required except a Postgres database.

## 1. Get a free Postgres database

Any of these work — pick one:

- **Supabase** → supabase.com → New project → Settings → Database → copy the
  "Connection string" (URI, use the "Transaction" pooler one for serverless).
- **Neon** → neon.tech → New project → copy the connection string.
- **Vercel Postgres** → from your Vercel dashboard → Storage → Create
  Database → Postgres.

You just need the resulting `DATABASE_URL`.

## Free APIs actually used (and ones that aren't really "free")

- **ip-api.com** — store region/city. **Free, no signup, no API key.** Used
  as-is in `src/lib/geo.ts`. Free-tier limit: 45 requests/minute per IP
  (fine for personal-scale scanning).
- **Shopify's own `/products.json`** — public by default, no key needed.
- **Pixel/theme/app detection** — custom regex over each store's own public
  homepage HTML (`src/lib/shopify.ts`), no third-party service at all.
- **Checked and ruled out as "best free API" candidates:** Wappalyzer and
  BuiltWith are the two biggest tech-detection databases, but neither has a
  genuinely free API for programmatic use — Wappalyzer's API starts at
  ~$149–250/month, BuiltWith's at ~$295/month; only their browser
  extensions / manual lookup pages are free. For the ~10 pixels/apps this
  project cares about, the custom detector above already covers it for $0.
- **Meta Ad Library API** — the one real, official, free API for ad data,
  but it requires *your own* Facebook Developer account + access token (no
  one can create that for you — it needs your login and, for broader
  access, identity verification). It only exposes ad creative/dates/
  platforms for ordinary commercial ads, never spend or impressions (those
  are restricted to political/social-issue ads by Meta's own policy). Not
  wired in yet — say the word and it gets added as an optional module once
  you have a token.

## Ad-channel detection

Each scan also reads the store's public homepage HTML and checks for Meta,
TikTok, Pinterest, and Snapchat pixel scripts — the same technique
BuiltWith-style tools use. This is a real signal (the store is literally
serving that script to every visitor), shown as badges on the **Stores**
page and filterable from the dashboard.

### About "sales" / "revenue" numbers

There's no free or paid API that gives real sales/revenue for a Shopify
store you don't own — that data is private to the merchant. Tools that show
a "monthly sales" number for a competitor's store are estimating it from
indirect signals (ppspy's own site says its algorithm "predicts" order
data). This project doesn't fabricate a number and label it as real. If you
later want an estimate, it would need to be built from a weak, clearly-
labeled proxy (review counts, "X sold" urgency widgets if the theme shows
them) — or from a paid data provider if you get access to one.

## 2. Local setup

```bash
npm install
cp .env.example .env
# edit .env: paste your DATABASE_URL, set CRON_SECRET to any random string
npx prisma db push      # creates/updates the tables in your database (safe to re-run anytime the schema changes)
npm run dev             # http://localhost:3000
```

Go to **Stores**, add a domain like `allbirds.com` or any Shopify store you
know, and it will scan immediately.

## 3. Deploy to Vercel

```bash
npm i -g vercel   # if you don't have the CLI
vercel             # follow the prompts, link/create the project
vercel env add DATABASE_URL production
vercel env add CRON_SECRET production
vercel --prod
```

(Or: push this folder to a GitHub repo and import it in the Vercel
dashboard, then add the same two env vars under Project Settings →
Environment Variables.)

## 4. Keep it updating automatically

`vercel.json` already defines a cron job that hits `/api/scrape` every 12
hours. Two things to know:

- Vercel's **Hobby (free) plan limits how often cron jobs can run** — check
  your current plan's limit in the Vercel dashboard (Settings → Cron Jobs)
  and adjust the `schedule` in `vercel.json` if needed; a daily run (e.g.
  `"0 6 * * *"`) is a safe default.
- You don't have to depend on cron at all: the **"🔄 Rescan all stores"**
  button on the dashboard, and the **"Rescan"** button per store, trigger a
  scan on demand, server-side, any time.

## Notes

- Scraping respects each store's own public JSON endpoint with a capped page
  count (politeness limit). A store that disables `/products.json` or blocks
  the request will show "Error" in the Stores table with the reason.
- This is for personal research use. Scrape at a reasonable pace and respect
  each store's terms of service.

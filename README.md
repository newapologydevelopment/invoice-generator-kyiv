# Invoice Builder · Генератор рахунків

Single-file bilingual invoice generator for Ukrainian sole proprietors (ФОП) billing US clients. No build step, no dependencies — open and use.

## Features

- Bilingual EN/UA invoice layout
- Amount in words («сума прописом»)
- Locale-aware currency formatting (USD, EUR, UAH)
- Auto-incrementing invoice numbers + local history
- Optional logo and contract reference
- Save to PDF via browser print
- Mobile-friendly edit / preview tabs

## Use locally

Open `index.html` in any modern browser. Business details persist in `localStorage`.

## Deploy

Static site — deploy the repo root to Vercel, Netlify, or GitHub Pages. `index.html` is the entry point.

```bash
vercel --prod
```

## Verify

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*?)<\/script>/);if(!m)throw new Error('no script');new Function(m[1]);console.log('SYNTAX OK')"
```

# Zadání: Instagram feed modul (Behold) do jiného projektu

Cílem je přidat **stejný modul s Instagram feedem**, jaký běží na webu Ubuntu Offline Festivalu.
Modul zobrazuje mřížku posledních příspěvků z Instagramu `@ubuntu.planet` v poměru **4:5**,
každá dlaždice odkazuje na konkrétní příspěvek a videa mají ikonu přehrávání.

---

## 1. Zdroj dat (Behold)

Feed nečteme přímo z Instagramu (to vyžaduje oficiální Graph API + schválení appky).
Používáme službu **[behold.so](https://behold.so)**, která má napojený účet a poskytuje veřejný JSON.

- **Feed JSON URL:** `https://feeds.behold.so/K4Fq1B5OLcBd1iFQrRn5`
- **Feed ID (token):** `K4Fq1B5OLcBd1iFQrRn5`
- **Instagram účet:** `https://www.instagram.com/ubuntu.planet`
- Endpoint je veřejný (GET, CORS povolený), **nepotřebuje žádný API klíč ani env proměnnou**.

### Struktura odpovědi (zkráceně)
```json
{
  "posts": [
    {
      "id": "...",
      "permalink": "https://www.instagram.com/p/...",
      "mediaType": "IMAGE" | "VIDEO" | "CAROUSEL_ALBUM",
      "mediaUrl": "https://...",
      "sizes": {
        "small":  { "mediaUrl": "...?class=squareSmall",  "width": 320, "height": 320 },
        "medium": { "mediaUrl": "...?class=squareMedium", "width": 640, "height": 640 },
        "large":  { "mediaUrl": "...?class=squareLarge",  "width": 800, "height": 800 }
      }
    }
  ]
}
```

### DŮLEŽITÉ — jak dostat 4:5 místo čtverce
Behold ve výchozích `sizes` vrací **čtvercový ořez** (`?class=squareLarge`).
Pro nezoříznutý originál (portrét) přepiš query parametr na **`?class=originalLarge`**:

```js
url.replace(/\?class=square[A-Za-z]+/, "?class=originalLarge")
```

Obrázek pak vsadíme do kontejneru s `aspect-ratio: 4 / 5` a `object-fit: cover`,
takže všechny dlaždice jsou jednotně 4:5.

---

## 2. Nastavení na straně Behold (nutné pro počet položek)

Kolik příspěvků feed vrátí, se **řídí v dashboardu behold.so**, ne v kódu.
Query parametry jako `?limit=12` endpoint ignoruje.

- Přihlas se na [behold.so](https://behold.so) → otevři tento feed.
- Nastav **Number of posts (`numPosts`)** na požadovaný počet (na webu chceme **12**).
- Účet `@ubuntu.planet` musí být na Instagramu typu **Professional / Business / Creator**,
  jinak Instagram Graph API feed nepovolí.

V kódu je pojistka `MAX_POSTS = 12` — vezme max. 12 položek, i kdyby feed vracel víc.

---

## 3. Chování modulu

- Mřížka dlaždic, každá **4:5**, zaoblené rohy `16px`, jemný stín.
- Řazení **chronologické** (nejnovější první — tak, jak přijdou z feedu).
- Responzivní počet sloupců:
  - `> 1000px`: **4** sloupce
  - `≤ 1000px`: **3** sloupce
  - `≤ 700px`: **2** sloupce
  - `≤ 460px`: **2** sloupce (menší mezera)
- Každá dlaždice = odkaz (`target="_blank" rel="noopener"`) na `post.permalink`.
- Video má v pravém horním rohu ikonu „play", obrázek ikonu Instagramu.
- Placeholder „Načítám…" před načtením; fallback hláška při chybě fetch.
- Lazy-loading obrázků (`loading="lazy"`), `crossorigin="anonymous"`.

---

## 4A. Implementace — čistý HTML / statická stránka

> Toto je 1:1 verze z festivalového webu. Vlož HTML tam, kam má modul patřit,
> a `<script>` na konec `<body>`.

### HTML (markup modulu)
```html
<section id="instagram" style="padding:clamp(56px,7vw,104px) 32px; background:#fdfbf7; border-top:1px solid rgba(32,30,29,.06)">
  <div style="max-width:1240px; margin:0 auto">
    <div style="position:relative; overflow:hidden; border-radius:28px; padding:clamp(40px,5vw,68px) clamp(28px,4vw,56px); background:linear-gradient(150deg,#fff4e7 0%,#fdece0 100%); border:1px solid rgba(32,30,29,.08); box-shadow:0 24px 60px -40px rgba(32,30,29,.4); text-align:center">
      <div style="position:absolute; top:-120px; right:-80px; width:360px; height:360px; border-radius:999px; background:radial-gradient(circle at 40% 40%, rgba(255,138,42,.22), rgba(255,255,255,0) 70%); pointer-events:none"></div>
      <div style="position:relative">
        <p style="display:inline-flex; align-items:center; gap:8px; font-size:12px; font-weight:800; letter-spacing:.1em; text-transform:uppercase; color:#a2521f; margin-bottom:16px">
          <span style="width:20px; height:2px; background:#ff5c14; border-radius:2px; display:block"></span>
          Zůstaň v obraze
        </p>
        <h2 style="font-size:clamp(30px,3.4vw,50px); line-height:1.06; margin-bottom:16px; text-wrap:balance">Více informací na našich<br>sociálních sítích</h2>
        <p style="font-size:16.5px; line-height:1.7; color:#4a4643; max-width:52ch; margin:0 auto clamp(28px,3.5vw,40px)">Program, novinky a dění z příprav festivalu sdílíme průběžně. Přidej se k nám a nic ti neuteče.</p>

        <style>
          @media (max-width: 1000px) { #ig-track { grid-template-columns:repeat(3, 1fr) !important; } }
          @media (max-width: 700px)  { #ig-track { grid-template-columns:repeat(2, 1fr) !important; } }
          @media (max-width: 460px)  { #ig-track { grid-template-columns:repeat(2, 1fr) !important; gap:10px !important; } }
        </style>
        <div id="ig-track" style="display:grid; grid-template-columns:repeat(4, 1fr); gap:14px; margin:0 auto clamp(28px,3.5vw,40px); max-width:1040px">
          <div style="aspect-ratio:4 / 5; border-radius:16px; background:rgba(32,30,29,.06); display:flex; align-items:center; justify-content:center; color:#8a857f; font-size:14px">Načítám…</div>
        </div>

        <div style="display:flex; flex-wrap:wrap; gap:12px; justify-content:center">
          <a href="https://www.instagram.com/ubuntu.planet" target="_blank" rel="noopener" style="display:inline-flex; align-items:center; gap:10px; font-size:15.5px; font-weight:700; padding:15px 28px; border-radius:999px; background:linear-gradient(180deg,#ff6a22,#e9541a); color:#fff; box-shadow:0 10px 22px -12px rgba(233,84,26,.9)">
            <svg width="19" height="19" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="2" y="2" width="20" height="20" rx="5" ry="5"></rect><path d="M16 11.37A4 4 0 1 1 12.63 8 4 4 0 0 1 16 11.37z"></path><line x1="17.5" y1="6.5" x2="17.51" y2="6.5"></line></svg>
            Instagram
          </a>
        </div>
      </div>
    </div>
  </div>
</section>
```

### JS (načtení feedu)
```html
<script>
(function () {
  var FEED = "https://feeds.behold.so/K4Fq1B5OLcBd1iFQrRn5";
  var MAX_POSTS = 12;
  var IG_URL = "https://www.instagram.com/ubuntu.planet";
  var track = document.getElementById("ig-track");
  if (!track) return;

  // Behold vrací čtvercový ořez (?class=squareLarge) – přepneme na nezoříznutý originál (4:5)
  function toPortrait(url) {
    return url ? url.replace(/\?class=square[A-Za-z]+/, "?class=originalLarge") : url;
  }

  function card(post) {
    var large = post.sizes && post.sizes.large ? post.sizes.large.mediaUrl : post.mediaUrl;
    var img = toPortrait(large) || post.mediaUrl;
    var isVideo = post.mediaType === "VIDEO";
    var a = document.createElement("a");
    a.href = post.permalink || IG_URL;
    a.target = "_blank";
    a.rel = "noopener";
    a.setAttribute("aria-label", "Otevřít příspěvek na Instagramu");
    a.style.cssText = "width:100%; aspect-ratio:4 / 5; border-radius:16px; overflow:hidden; position:relative; background:#e9e4dc; display:block; box-shadow:0 12px 28px -18px rgba(32,30,29,.55)";
    a.innerHTML =
      '<img src="' + img + '" alt="Příspěvek na Instagramu" loading="lazy" crossorigin="anonymous" style="width:100%; height:100%; object-fit:cover; display:block">' +
      '<span style="position:absolute; inset:0; background:linear-gradient(180deg, rgba(32,30,29,0) 55%, rgba(32,30,29,.55) 100%)"></span>' +
      '<span style="position:absolute; top:10px; right:10px; width:26px; height:26px; border-radius:999px; background:rgba(255,255,255,.9); display:flex; align-items:center; justify-content:center; color:#201e1d">' +
        (isVideo
          ? '<svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><polygon points="5 3 19 12 5 21 5 3"></polygon></svg>'
          : '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="2" y="2" width="20" height="20" rx="5" ry="5"></rect><path d="M16 11.37A4 4 0 1 1 12.63 8 4 4 0 0 1 16 11.37z"></path><line x1="17.5" y1="6.5" x2="17.51" y2="6.5"></line></svg>') +
      '</span>';
    return a;
  }

  fetch(FEED)
    .then(function (r) { return r.json(); })
    .then(function (data) {
      var posts = ((data && data.posts) || []).slice(0, MAX_POSTS);
      if (!posts.length) throw new Error("no posts");
      track.innerHTML = "";
      posts.forEach(function (p) { track.appendChild(card(p)); });
    })
    .catch(function () {
      track.innerHTML =
        '<div style="grid-column:1 / -1; text-align:center; color:#8a857f; font-size:14.5px; padding:24px">Příspěvky se teď nepodařilo načíst — mrkni přímo na náš <a href="' + IG_URL + '" target="_blank" rel="noopener" style="color:#e9541a; font-weight:700">Instagram</a>.</div>';
    });
})();
</script>
```

> **Poznámka:** Na původním festivalovém webu je navíc `setInterval` retry-smyčka, protože ten
> web běží přes framework `x-dc`, který po skriptu překresluje DOM a přepsal by vložené karty.
> **V běžném projektu (React/Next/Vite/statické HTML) tuto smyčku NEPOTŘEBUJEŠ** — použij verzi výše.

---

## 4B. Implementace — React / Next.js (doporučeno pro moderní projekt)

Komponenta `InstagramFeed.tsx` (client component). Používá `fetch` v `useEffect`.

```tsx
"use client";

import { useEffect, useState } from "react";

const FEED_URL = "https://feeds.behold.so/K4Fq1B5OLcBd1iFQrRn5";
const IG_URL = "https://www.instagram.com/ubuntu.planet";
const MAX_POSTS = 12;

type BeholdPost = {
  id: string;
  permalink?: string;
  mediaType?: "IMAGE" | "VIDEO" | "CAROUSEL_ALBUM";
  mediaUrl?: string;
  sizes?: { large?: { mediaUrl: string } };
};

// Behold vrací čtvercový ořez – přepneme na nezoříznutý originál (4:5)
function toPortrait(url?: string) {
  return url ? url.replace(/\?class=square[A-Za-z]+/, "?class=originalLarge") : url;
}

export function InstagramFeed() {
  const [posts, setPosts] = useState<BeholdPost[] | null>(null);
  const [error, setError] = useState(false);

  useEffect(() => {
    let active = true;
    fetch(FEED_URL)
      .then((r) => r.json())
      .then((data) => {
        if (!active) return;
        const list: BeholdPost[] = (data?.posts ?? []).slice(0, MAX_POSTS);
        if (!list.length) throw new Error("no posts");
        setPosts(list);
      })
      .catch(() => active && setError(true));
    return () => { active = false; };
  }, []);

  return (
    <section className="ig-section">
      <div className="ig-inner">
        <p className="ig-eyebrow">Zůstaň v obraze</p>
        <h2 className="ig-title">Více informací na našich sociálních sítích</h2>
        <p className="ig-lead">Program, novinky a dění z příprav sdílíme průběžně. Přidej se k nám a nic ti neuteče.</p>

        <div className="ig-grid">
          {error && (
            <div className="ig-error">
              Příspěvky se teď nepodařilo načíst — mrkni přímo na náš{" "}
              <a href={IG_URL} target="_blank" rel="noopener noreferrer">Instagram</a>.
            </div>
          )}
          {!error && !posts && <div className="ig-skeleton">Načítám…</div>}
          {!error && posts?.map((post) => {
            const src = toPortrait(post.sizes?.large?.mediaUrl ?? post.mediaUrl) ?? post.mediaUrl;
            const isVideo = post.mediaType === "VIDEO";
            return (
              <a
                key={post.id}
                href={post.permalink ?? IG_URL}
                target="_blank"
                rel="noopener noreferrer"
                aria-label="Otevřít příspěvek na Instagramu"
                className="ig-card"
              >
                <img src={src} alt="Příspěvek na Instagramu" loading="lazy" crossOrigin="anonymous" />
                <span className="ig-overlay" />
                <span className="ig-badge">{isVideo ? "▶" : "◎"}</span>
              </a>
            );
          })}
        </div>

        <a href={IG_URL} target="_blank" rel="noopener noreferrer" className="ig-btn">Instagram</a>
      </div>
    </section>
  );
}
```

```css
/* InstagramFeed.css – uprav barvy dle vlastního webu */
.ig-section { padding: clamp(56px,7vw,104px) 32px; background: #fdfbf7; }
.ig-inner { max-width: 1240px; margin: 0 auto; text-align: center; }
.ig-eyebrow { font-size: 12px; font-weight: 800; letter-spacing: .1em; text-transform: uppercase; color: #a2521f; margin-bottom: 16px; }
.ig-title { font-size: clamp(30px,3.4vw,50px); line-height: 1.06; margin-bottom: 16px; text-wrap: balance; }
.ig-lead { font-size: 16.5px; line-height: 1.7; color: #4a4643; max-width: 52ch; margin: 0 auto clamp(28px,3.5vw,40px); }

.ig-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 14px;
  max-width: 1040px;
  margin: 0 auto clamp(28px,3.5vw,40px);
}
@media (max-width: 1000px) { .ig-grid { grid-template-columns: repeat(3, 1fr); } }
@media (max-width: 700px)  { .ig-grid { grid-template-columns: repeat(2, 1fr); } }
@media (max-width: 460px)  { .ig-grid { grid-template-columns: repeat(2, 1fr); gap: 10px; } }

.ig-card {
  position: relative;
  display: block;
  width: 100%;
  aspect-ratio: 4 / 5;
  border-radius: 16px;
  overflow: hidden;
  background: #e9e4dc;
  box-shadow: 0 12px 28px -18px rgba(32,30,29,.55);
}
.ig-card img { width: 100%; height: 100%; object-fit: cover; display: block; }
.ig-overlay { position: absolute; inset: 0; background: linear-gradient(180deg, rgba(32,30,29,0) 55%, rgba(32,30,29,.55) 100%); }
.ig-badge { position: absolute; top: 10px; right: 10px; width: 26px; height: 26px; border-radius: 999px; background: rgba(255,255,255,.9); display: flex; align-items: center; justify-content: center; color: #201e1d; font-size: 13px; }

.ig-skeleton { aspect-ratio: 4 / 5; border-radius: 16px; background: rgba(32,30,29,.06); display: flex; align-items: center; justify-content: center; color: #8a857f; font-size: 14px; }
.ig-error { grid-column: 1 / -1; text-align: center; color: #8a857f; font-size: 14.5px; padding: 24px; }
.ig-error a { color: #e9541a; font-weight: 700; }

.ig-btn {
  display: inline-flex; align-items: center; gap: 10px;
  font-size: 15.5px; font-weight: 700; padding: 15px 28px; border-radius: 999px;
  background: linear-gradient(180deg,#ff6a22,#e9541a); color: #fff;
  box-shadow: 0 10px 22px -12px rgba(233,84,26,.9); text-decoration: none;
}
```

> V Reactu můžeš `▶` / `◎` v `.ig-badge` nahradit ikonami z `lucide-react`
> (`<Play />` / `<Instagram />`) — logika zůstává stejná.

---

## 5. Volitelně: proxy přes vlastní API (odolnější)

Pokud nechceš volat Behold z klienta (kvůli caching/CORS/skrytí feedu),
udělej server route (Next.js) a cachuj odpověď:

```ts
// app/api/instagram/route.ts
export const revalidate = 900; // 15 min cache

export async function GET() {
  const res = await fetch("https://feeds.behold.so/K4Fq1B5OLcBd1iFQrRn5", {
    next: { revalidate: 900 },
  });
  const data = await res.json();
  return Response.json({ posts: (data?.posts ?? []).slice(0, 12) });
}
```
Komponenta pak volá `/api/instagram` místo přímé Behold URL.

---

## 6. Akceptační kritéria

- [ ] Modul načte a zobrazí příspěvky z `@ubuntu.planet` přes feed `K4Fq1B5OLcBd1iFQrRn5`.
- [ ] Dlaždice jsou v poměru **4:5** (ne čtverec) — použit `?class=originalLarge`.
- [ ] Počet položek řízen Beholdem (`numPosts`), v kódu strop `MAX_POSTS = 12`.
- [ ] Responzivně 4 / 3 / 2 sloupce dle šířky.
- [ ] Každá dlaždice odkazuje na svůj `permalink`, otevírá se v nové kartě.
- [ ] Videa mají play ikonu, funguje placeholder „Načítám…" i chybový fallback.
- [ ] Žádný API klíč ani env proměnná nejsou potřeba (endpoint je veřejný).
```

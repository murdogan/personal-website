# Faz 1 — Statik Site, Testler ve Dağıtım Hattı

**Amaç:** Yer tutucu içerikle çalışan, testleri ve kalite kapıları olan statik bir site; onu sunucuya güvenli biçimde dağıtan `deployctl` ve CI/CD hattı.
**Önkoşul:** Faz 0 kapısı.
**Referans:** V2 §5 (depo), §6 (içerik), §7 (sürüm hattı ve 7.1), §12 (frontend), §14 (önbellek), §15 (güvenlik), §16 (loglar), §18 (dağıtım).
**Kapsam dışı:** Arama (Faz 2), nihai görsel tasarım ve animasyon (Faz 3), asistan (Faz 4), sıkı CSP (Faz 5).

Bu fazın sonunda site, `SITE_MODE=prelaunch` (arama motorlarına kapalı) olarak sunucuda yayında olur. Sunucu hazır değilse İP-1.11'in sunucu adımları ve İP-1.13 ertelenir; **kod kapısı** geçilir, diğer fazlar beklemez (README §4).

**Komut kuralları (bu fazdan itibaren):**

- Rust komutları depo kökünden `--manifest-path backend/Cargo.toml` ile çalışır; `just` recipe'leri bunu yapar. Göreli yollar (`config/`, `out/`, `content/`) depo köküne göredir.
- Ortam değişkeni gereken recipe'ler `just`'ın `$` önekli parametreleriyle yazılır (ör. `build $SITE_MODE="prelaunch":`). `VAR=x komut` biçimi PowerShell'de çalışmaz ve kullanılmaz.
- Belgelerdeki `curl` örnekleri Windows'ta `curl.exe` ile çalıştırılır; JSON gövdesi dosyadan verilir (`--data-binary @dosya.json`).

---

## İP-1.1 — Depo iskeleti ve araç zinciri

**Amaç:** Bütün İP'lerin üzerine kurulacağı depo düzeni ve ortak komutlar.

**Yapılacaklar:**

1. Kök dosyalar:
   - `.gitignore`: `node_modules/`, `target/`, `dist/`, `.astro/`, `out/`, `.env*` (`.env.example` hariç), `test-results/`, `playwright-report/`, `.lighthouseci/`.
   - `.gitattributes`: `* text=auto eol=lf`; görseller ve fontlar (`*.png *.jpg *.jpeg *.webp *.avif *.pdf *.woff2`) `binary`.
   - `.editorconfig`: UTF-8, LF, 2 boşluk (Rust için 4).
   - `README.md`: proje özeti, belge haritası (AGENTS.md'ye bağlantı), hızlı başlangıç (`just setup`, `just dev`, `just check`).
2. Node tarafı:
   - Kök `package.json` (`private: true`), `packageManager: "pnpm@<sabit sürüm>"`, `engines.node`.
   - `pnpm-workspace.yaml` → `packages: ["web"]`.
   - `.nvmrc` (Node LTS sürümü).
3. Rust tarafı: yalnız kökte `rust-toolchain.toml` (güncel kararlı sürüm **numarasıyla**, `components = ["rustfmt", "clippy"]`). Cargo workspace ilk crate ile birlikte İP-1.8'de oluşturulur (üyesiz workspace derlenmez).
4. `justfile`:
   - `set windows-shell := ["powershell.exe", "-NoLogo", "-NoProfile", "-Command"]`.
   - Recipe'ler: `setup`, `fmt`, `check`, `test`, `build`. Henüz var olmayan parçalar için recipe "henüz yok" yazıp başarıyla çıkar.
   - Recipe'ler tek komut çağırır; mantık betiklerdedir. Komut kuralları yukarıda.
5. `docs/adr/`: V2 §2.2'deki ADR-01…ADR-08 için birer kısa dosya (`0001-icerik-git.md` …). Biçim: Bağlam, Karar, Sonuçlar, Yeniden değerlendirme koşulu. İçerik V2'den alınır; yeni karar eklenmez. ADR numaraları README §4'teki tabloya uyar.
6. Mevcut belgeler (`PROJE_KAPSAMI.md`, `TASARIM.md`, `TEKNIK_MIMARI*.md`, `referanslar/`) yerinde kalır.

**Bitti sayılır:**

- [ ] `just --list` bütün recipe'leri gösteriyor.
- [ ] `just setup` ve `just check` Windows'ta hatasız tamamlanıyor.
- [ ] `docs/adr/` altında 8 ADR var.

---

## İP-1.2 — Astro uygulaması ve kalite araçları

**Amaç:** Boş ama yapılandırılmış Astro projesi.

**Yapılacaklar:**

1. `web/` altında Astro projesi: TypeScript `strictest`; entegrasyonlar `@astrojs/react`, `@astrojs/mdx`, `@astrojs/sitemap`; `sharp` doğrudan bağımlılık (pnpm altında görsel hattı için gerekir); `@types/react`, `@types/react-dom`, `@types/node`.
2. `web/astro.config.mjs`:
   - `site`: `SITE_URL` ortam değişkeni (varsayılan `http://localhost:4321`).
   - `output: "static"`, `trailingSlash: "never"`, `build: { format: "file" }`.
   - Ortam şeması (`astro:env`): `SITE_RELEASE` (string, varsayılan `dev`), `SITE_MODE` (`prelaunch` | `public`, varsayılan `prelaunch`); derleme zamanında okunur, public erişimli. Desteklenmiyorsa `import.meta.env` + zod doğrulaması.
   - Vite dev sunucusunda `/api` → `http://127.0.0.1:8080` proxy (Faz 2B kullanır).
3. Ortam okuma tek noktadan:
   - Astro kodu: `web/src/lib/site.ts` (`SITE_RELEASE`, `SITE_MODE`, `isPublicMode()`).
   - Node betikleri (`web/scripts/`, `tsx` ile; `astro:env`'i içe aktaramazlar): `web/scripts/lib/env.ts` (`process.env` + aynı doğrulama).
   - Başka yer ortam değişkenini doğrudan okumaz.
4. Kalite araçları: ESLint flat config (typescript-eslint, eslint-plugin-astro, eslint-plugin-react-hooks, eslint-plugin-jsx-a11y); Prettier + prettier-plugin-astro. `web/package.json` scriptleri: `dev`, `build`, `preview`, `check` (`astro check`), `lint`, `format`, `format:check`, `test` (vitest), `e2e` (playwright).
5. Vitest (`web/vitest.config.ts`) ve örnek test (`src/lib/site.test.ts`).
6. `justfile`: `check` → `astro check`, lint, format kontrolü, vitest; `build $SITE_MODE="prelaunch" $SITE_RELEASE="dev"` → web derlemesi.

**Bitti sayılır:**

- [ ] `just build` → `web/dist/index.html` üretiliyor.
- [ ] `astro check`, ESLint, Prettier temiz; `just check` geçiyor.

---

## İP-1.3 — İçerik şemaları, doğrulama ve örnek içerik

**Amaç:** İçeriğin tek kaynağı ve onu koruyan şema. Referans: V2 §6.

**Yapılacaklar:**

1. `content/` yapısı:

   ```text
   content/
     profile.md
     portrait-placeholder.png           yer tutucu siluet (alfa kanallı)
     tags.yaml
     projects/
       ornek-proje-1/index.mdx, kapak.jpg
       ornek-proje-2/index.mdx, kapak.jpg
       ornek-proje-3/index.md,  kapak.jpg
   ```

2. Yer tutucu görseller `web/scripts/make-placeholders.ts` (sharp, SVG'den) ile üretilir ve commit'lenir: kapaklar 1600×1000 gri tonlu, üzerinde "Örnek Proje N"; portre 1200×1600 şeffaf arka planlı nötr siluet.
3. `web/src/content.config.ts`:
   - `profile`: `glob({ pattern: "profile.md", base: "../content" })`. Şema: `name` (1–60), `headline` (1–80), `shortBio` (1–300), `siteLabel` (≤ 40), `heroLines` (string[], 1–4 öğe, her biri ≤ 40), `projectsEyebrow` (≤ 40), `projectsHeading` (≤ 80), `skills` (string[]), `links`: `email` (zorunlu), `github?`, `linkedin?`, `cv?` (URL veya site içi yol), `portrait`: `{ src: image(), alt: string(min 1) }`, `placeholder` (boolean, varsayılan false).
   - `projects`: `glob({ pattern: "*/index.{md,mdx}", base: "../content/projects" })`; kimlik klasör adıdır (slug). Şema: `title` (1–80), `summary` (1–160), `role` (1–120), `tags` (`reference("tags")` dizisi, en az 1), `year` (2000–2100), `status` (`tamamlandi` | `suruyor` | `arsiv`), `featured` (varsayılan false), `order` (varsayılan 100), `cover`: `{ src: image(), alt: string(min 1) }`, `repo?` (URL), `demo?`: `{ kind: "static" | "service", url: URL }`, `draft` (varsayılan false), `placeholder` (varsayılan false).
   - `tags`: `file("../content/tags.yaml")`. Şema: `id` (`^[a-z0-9]+(-[a-z0-9]+)*$`), `label` (1–40), `aliases` (string[]), `group` (`dil` | `framework` | `veri` | `arac` | `platform` | `alan`).
   - Glob tabanının proje kökü dışını (`../content`) desteklediği kurulu Astro sürümünde doğrulanır; desteklemiyorsa durulur ve sorulur.
4. `content/tags.yaml` başlangıç sözlüğü: en az `csharp` (C#), `dotnet` (.NET), `aspnetcore` (ASP.NET Core), `rust`, `typescript`, `javascript`, `react`, `astro`, `nodejs` (Node.js), `cpp` (C++), `postgresql`, `docker`, `linux`. `asp.net core` ve `asp.net` eşanlamlıları yalnız `aspnetcore`'dadır.
5. Örnek içerik: `profile.md` ve 3 proje; hepsi `placeholder: true`; metinler açıkça yer tutucu ("Ad Soyad", "Örnek Proje 1", "Bu bir yer tutucu açıklamadır."). Proje gövdeleri `## Problem`, `## Çözüm`, `## Katkım`, `## Teknik notlar` başlıklarını içerir. Bir proje `demo`, biri `repo` alanına sahip; biri `draft: true`.
6. **Yer tutucu koruması derlemenin kendisindedir:** `web/src/lib/content.ts` yardımcıları `SITE_MODE=public` iken `placeholder: true` bir girdiyle karşılaşırsa hata fırlatır; derleme durur. Böylece `just check` atlanan bir yayında da yer tutucu yayımlanamaz.
7. Ek doğrulama betiği `web/scripts/check-content.ts` (`tsx`; `--mode prelaunch|public`; `just check` ve `just check-public` çağırır). Zod'un yakalayamadıklarını denetler. Front-matter `---` sınırlarından ayrılıp `yaml` ile, `tags.yaml` `yaml` ile okunur; görsel boyutları `sharp` ile:
   - Proje klasör adı `^[a-z0-9]+(-[a-z0-9]+)*$`.
   - Görsel dosyası ≤ 5 MB; kapak en az 1200 px genişlik (önerilen 1600).
   - Etiket eşanlamlıları Türkçe büyük/küçük harf duyarsız olarak bütün etiketler arasında tekil; bir eşanlamlı başka bir etiketin `label`'ı ile çakışmaz.
   - `--mode public` iken `placeholder: true` içerik → hata.
   - Hatalar `dosya: alan — açıklama` biçiminde; çıkış kodu 1.
8. `web/src/lib/content.ts` (tek giriş noktası; sayfalar `getCollection`'ı doğrudan çağırmaz):
   - `getProfile()`; `getPublishedProjects()`: üretim derlemesinde `draft` hariç; sıralama `featured` (önce), `order` (artan), `year` (azalan), `slug` (artan).
   - `getFeaturedProjects()`, `getProjectsByTag(tagId)`, `getUsedTags()` (yalnız yayımlanmış projelerde kullanılanlar; `Intl.Collator("tr")` ile sıralı).
9. `justfile`: `check-public` (= `check-content --mode public` + `SITE_MODE=public` derleme denemesi, `$` parametreleriyle).
10. Vitest: sıralama, draft filtresi, public kipte yer tutucu hatası, `check-content` kurallarının her biri. Fixture'lar `web/tests/fixtures/content/`.

**Bitti sayılır:**

- [ ] Geçersiz etiket referansı, eksik `alt` veya 161 karakterlik özet derlemeyi durduruyor (testle).
- [ ] `just check-public` yer tutucu içerik nedeniyle başarısız; `just check` geçiyor.
- [ ] Bir `.mdx` proje sayfası üretiliyor (MDX entegrasyonu çalışıyor).

---

## İP-1.4 — Sayfa iskeleti, rotalar ve düzen bileşenleri

**Amaç:** Bütün sayfaların anlamlı HTML ile var olması. Görsel cila Faz 3'tedir.

**Rotalar:**

| Yol | Dosya | İçerik |
| --- | --- | --- |
| `/` | `pages/index.astro` | Hero (portre, isim, `siteLabel`, `heroLines`, `#calismalar` bağlantısı), seçili çalışmalar (`id="calismalar"`) |
| `/projeler` | `pages/projeler/index.astro` | Bütün projeler + etiket filtre bağlantıları |
| `/projeler/<slug>` | `pages/projeler/[slug].astro` | Proje detayı |
| `/projeler/etiket/<tag>` | `pages/projeler/etiket/[tag].astro` | Etikete göre projeler (yalnız kullanılan etiketler) |
| `/hakkimda` | `pages/hakkimda.astro` | Hakkımda gövdesi, yetenekler, CV |
| `/ara` | `pages/ara.astro` | Arama sonuç sayfasının kabuğu: `<noscript>` mesajı + etiket bağlantıları (ada Faz 2B'de) |
| 404 | `pages/404.astro` | Mesaj, arama formu, projeler bağlantısı |

**Yol yardımcısı:** `build.format: "file"` derlemede `Astro.url.pathname`'i `/projeler.html` biçiminde verir. Bu yüzden `web/src/lib/paths.ts` → `pagePath(url)` (`.html` ve `/index` atılır, kök `/`). Kanonik URL, `og:url`, JSON-LD `url`, `aria-current` karşılaştırmaları **yalnız** bu yardımcıyı kullanır.

**Bileşenler** (`web/src/components/`):

- `layout/BaseLayout.astro`: `<html lang="tr">`; `<head>`: charset, viewport, `<title>` ("Sayfa — {name}"; ana sayfada `{name} — {headline}`), description, kanonik (`SITE_URL` + `pagePath`), `<meta name="release" content={SITE_RELEASE}>`, robots meta (İP-1.6), Open Graph temel etiketleri, `head` slot'u. `<body>`: "İçeriğe geç" atlama bağlantısı (`#main`), `SiteHeader`, `<main id="main" tabindex="-1">`, `SiteFooter`.
- `layout/SiteHeader.astro`: isim (ana sayfaya), gezinme (`Projeler` → `/projeler`, `Hakkımda` → `/hakkimda`, `İletişim` → `#iletisim`), arama formu. Etkin sayfa `aria-current="page"` (`pagePath` ile).
- `search/SearchForm.astro`: `<form role="search" action="/ara" method="get">`, görsel olarak gizli ama erişilebilir etiket ("Sitede ara"), `<input type="search" name="q" placeholder="Sitede ara…" autocomplete="off" enterkeyhint="search">`. Sarmalayıcı `data-search-root` (Faz 2B). JS olmadan `/ara?q=…`'ya gider.
- `layout/SiteFooter.astro`: `id="iletisim"`, başlık "İletişime geç.", e-posta (`mailto:`), GitHub, LinkedIn, CV (yalnız tanımlı olanlar).
- `project/ProjectRow.astro`: kapak (`<Image>`, `width`/`height`), başlık (detaya), özet, etiketler, "Projeyi incele", "Canlı dene" (yalnız `demo` varsa; yeni sekme, `rel="noopener noreferrer"`, görünür "yeni sekmede açılır" ipucu).
- `project/TagList.astro`: etiket bağlantıları.
- `mdx/`: `Figure`, `Callout`; `<Content components={…}>` ile verilir.

**Proje detayı:** başlık (`h1`), özet, rol, yıl, durum, etiketler, kapak, gövde (`render(entry)`), "Kaynak kod" ve "Canlı dene", önceki/sonraki proje.

**Kurallar:** Her sayfada tek `h1`; başlık hiyerarşisi atlamasız. Portre `<Picture>` (AVIF + WebP), `loading="eager"`, `fetchpriority="high"`, açık boyutlar. Diğer görseller `loading="lazy"`. Henüz ada yok; sayfalar sıfır JS.

**Testler:**

- `web/scripts/check-links.ts`: `dist/` içindeki bütün HTML'lerin site içi `href`'lerini (anchor'lar dahil) doğrular; kırıkta çıkış 1. `just check` derlemeden sonra çalıştırır.
- `dist/` üzerinde test: kanonik URL'lerde ve `aria-current`'ta `.html` yok; her sayfanın kanonik değeri beklenen yol.

**Bitti sayılır:**

- [ ] Bütün rotalar derleniyor; draft proje üretim derlemesinde yok, geliştirmede var.
- [ ] Bağlantı ve kanonik URL denetimleri temiz.
- [ ] JS kapalıyken bütün sayfalar okunabilir ve gezilebilir (elle kontrol).

---

## İP-1.5 — Temel stil katmanı ve token'lar

**Amaç:** Okunaklı bir temel. Nihai tasarım İP-3.1'de kesinleşir.

**Yapılacaklar:**

1. `web/src/styles/tokens.css` (`:root`). Renkler tasarım taslaklarından ölçülmüştür (A-08 onayına tabi):

   | Token | Değer | Kullanım |
   | --- | --- | --- |
   | `--color-ink-950` | `#111414` | Koyu zemin |
   | `--color-ink-900` | `#1a1d21` | Açık zeminde başlık/metin |
   | `--color-ink-700` | `#303030` | Koyu zeminde ayırıcı |
   | `--color-paper-50` | `#f8f4ec` | Sıcak beyaz: açık zemin, koyu zeminde metin |
   | `--color-paper-200` | `#e3dfd8` | Açık zeminde ayırıcı |
   | `--color-stone-600` | `#6b6b66` | Açık zeminde ikincil metin (≈ 4,9:1) |
   | `--color-stone-400` | `#9a9a95` | Koyu zeminde ikincil metin |
   | `--color-accent-400` | `#cfe16c` | Sarı-yeşil vurgu, yumuşak (düğmeler, simgeler) |
   | `--color-accent-300` | `#c8f050` | Sarı-yeşil vurgu, canlı (asistan simgesi, akış çizgileri) |

   Vurgu renkleri yalnız koyu zeminde kullanılır; açık zeminde metin rengi olamaz. Ayrıca: boşluk ölçeği (`--space-1` … `--space-10`), akışkan yazı ölçeği (`clamp()`), satır yükseklikleri, `--radius-pill: 999px`, `--border-hairline: 1px`, hareket süreleri ve eğrileri.
2. Tema: `[data-theme="dark"]` (zemin ink-950, metin paper-50) ve `[data-theme="light"]` (zemin paper-50, metin ink-900).
3. `web/src/styles/global.css`: modern reset, temel tipografi, `:focus-visible` halkası (iki temada ≥ 3:1), `.visually-hidden`, `prefers-reduced-motion` global kuralı.
4. Yazı tipi: `@fontsource-variable/inter` (A-07 varsayılanı), latin + latin-ext.
5. Düzen: içerik genişliği (`--container-max`), yatay kenar boşluğu (mobilde 16–20 px).

**Bitti sayılır:**

- [ ] Ana sayfa, liste ve detay masaüstü ve 375 px'te düzgün; yatay kaydırma yok (elle).
- [ ] Kontrastlar AA (axe ile İP-1.7'de).

---

## İP-1.6 — SEO ve site kipi (prelaunch/public)

**Yapılacaklar:**

1. `prelaunch`: her sayfada `<meta name="robots" content="noindex, nofollow">`; `robots.txt` → `Disallow: /`.
2. `public`: robots meta yok; `robots.txt` → `Allow: /` + `Sitemap: <SITE_URL>/sitemap-index.xml`.
3. `robots.txt` endpoint ile (`pages/robots.txt.ts`).
4. Sitemap: `/ara`, `/404`, `/internal/*` hariç; URL'ler `pagePath` biçiminde.
5. JSON-LD: ana sayfa ve `/hakkimda`'da `Person`; proje detayında `CreativeWork`. `placeholder: true` içerik için üretilmez.
6. Open Graph: `og:title`, `og:description`, `og:url`, `og:type`, `og:image` (projede kapaktan 1200×630 JPEG; diğer sayfalarda `public/og-default.png` yer tutucu), `og:locale = tr_TR`.

**Testler:** robots içeriği iki kipte; JSON-LD çıktısı; `prelaunch` derlemesinde her HTML'de noindex.

**Bitti sayılır:**

- [ ] Testler geçiyor; sitemap'te `/ara` ve `/404` yok.

---

## İP-1.7 — Caddy yapılandırması ve frontend testleri

**Amaç:** Testlerin üretimdeki Caddy davranışıyla (yönlendirme, `try_files`, 404, başlıklar, sıkıştırma) çalışması. Yönlendirme yapılandırması **sürüm paketinin parçasıdır** (İP-1.8); böylece depo değişiklikleri her dağıtımla sunucuya ulaşır ve geri almada eski yapılandırma da geri gelir.

**Yapılacaklar:**

1. `deploy/caddy/security-headers.caddy` (argümansız içe aktarılan dosya):

   ```caddy
   header {
   	Strict-Transport-Security "max-age=31536000; includeSubDomains"
   	X-Content-Type-Options "nosniff"
   	Referrer-Policy "strict-origin-when-cross-origin"
   	Permissions-Policy "camera=(), microphone=(), geolocation=()"
   	Cross-Origin-Opener-Policy "same-origin"
   	Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'"
   	-Server
   }
   ```

   `script-src 'unsafe-inline'` geçicidir; İP-5.1'de sürüm başına hash'lerle değiştirilir.
2. `deploy/caddy/site-routes.caddy` (argümanla içe aktarılan dosya: `import <dosya> <statik-kök> <varlık-kökü> <api-upstream>`). Taslak; sözdizimi kurulu Caddy sürümünde `caddy validate` ile doğrulanır:

   ```caddy
   encode zstd gzip

   handle /internal/* {
   	respond 404
   }

   handle /api/* {
   	request_body {
   		max_size 32KB
   	}
   	reverse_proxy {args[2]}
   }

   handle /_astro/* {
   	root * {args[1]}
   	header Cache-Control "public, max-age=31536000, immutable"
   	file_server
   }

   handle {
   	root * {args[0]}
   	header Cache-Control "no-cache"
   	try_files {path} {path}.html {path}/index.html
   	file_server
   }

   handle_errors {
   	@404 expression {err.status_code} == 404
   	handle @404 {
   		root * {args[0]}
   		rewrite * /404.html
   		file_server
   	}
   }
   ```

   - Caddy gövde sınırı (32 KB) API'ninkinden (16 KB) büyüktür; böylece normal aşımda API'nin problem+json yanıtı döner, Caddy yalnız yedek sınırdır.
   - `flush_interval` **kullanılmaz** (AGENTS.md §10).
   - `/internal/*` ayrı `handle` bloğuyla kapatılır (`respond`'u tümünü yakalayan `handle`'ın dışında bırakmak çalışmaz).
3. `deploy/caddy/Caddyfile.local` (yerel ve CI testleri): `admin off`, `auto_https off`; `http://localhost:8088` bloğu; `import security-headers.caddy`; `import site-routes.caddy {$SITE_DIST} {$SITE_DIST} {$SITE_API:127.0.0.1:8080}` (yerelde `_astro` doğrudan `dist`'ten). Göreli `import` yolunun neye göre çözüldüğü doğrulanır; gerekirse mutlak yol ortam değişkeniyle verilir. Bu dosya log yazmaz.
4. Playwright (`web/playwright.config.ts`): `webServer` Caddy'yi `Caddyfile.local` ile başlatır, `baseURL: http://localhost:8088`. `just e2e` önce derler. Projeler: masaüstü ve mobil Chromium; JS kapalı bağlam ayrı dosyada.
5. Testler (`web/tests/e2e/`):
   - `smoke.spec.ts`: ana sayfa `h1` ve portre; gezinme; liste → detay → etiket; 404 sayfası 404 durum koduyla; `/projeler` uzantısız URL.
   - `nojs.spec.ts`: JS kapalıyken içerik okunuyor, gezinme ve arama formu (`/ara?q=…`) çalışıyor.
   - `a11y.spec.ts`: axe ile ana sayfa, liste, detay, hakkımda; `serious`/`critical` ihlal yok.
   - `keyboard.spec.ts`: atlama bağlantısı odağı `#main`'e taşıyor; görünür odak.
   - `headers.spec.ts`: HTML `no-cache`; `/_astro/*` `immutable`; güvenlik başlıkları; `/internal/x` 404.
6. Lighthouse CI (`web/lighthouserc.cjs`): Caddy üzerinden `/`, `/projeler`, bir proje detayı; mobil; 3 koşu. Assertion'lar: LCP ≤ 2500 ms, CLS ≤ 0.1, TBT ≤ 200 ms, `resource-summary:script:size` ≤ 102400 bayt, erişilebilirlik ≥ 0.95; `is-crawlable` kapalı (prelaunch).
7. `justfile`: `e2e`, `lighthouse`.

**Bitti sayılır:**

- [ ] `caddy validate --config deploy/caddy/Caddyfile.local --adapter caddyfile` başarılı.
- [ ] `just e2e` ve `just lighthouse` yerelde geçiyor.

---

## İP-1.8 — deployctl: workspace, paket, durum, kilit, varlık deposu

**Amaç:** Sunucudaki sürüm işlemlerini yapan, test edilebilir Rust CLI'nin çekirdeği. Referans: V2 §7, §7.1, §18.2; D-10, D-11, D-12. Projenin ilk Rust kodu: senkron (async yok), dosya I/O, serde, tipli hatalar.

**Workspace (bu İP'de oluşturulur):**

- `backend/Cargo.toml`: `resolver = "3"`, `members = ["crates/deployctl"]`, `[workspace.package] edition = "2024"`, `[workspace.lints]` (AGENTS.md §8.2), `[workspace.dependencies]`.
- `backend/clippy.toml`: `allow-unwrap-in-tests`, `allow-expect-in-tests`, `allow-panic-in-tests` = true.
- `backend/rustfmt.toml`: `edition = "2024"`.
- `backend/deny.toml`: izinli lisanslar MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, Unicode-3.0, Zlib, MPL-2.0, CC0-1.0, **CDLA-Permissive-2.0** (rustls kök sertifika paketi `webpki-roots` bu lisansı kullanır; önceden onaylıdır). Güvenlik bildirimi `deny`.
- `justfile` `check`'e Rust adımları: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, `cargo deny check` (hepsi `--manifest-path backend/Cargo.toml`).

**Crate:** `backend/crates/deployctl` (bin `deployctl`). Bağımlılıklar: clap, serde, serde_json, toml, thiserror, anyhow, time, tar, flate2, sha2, blake3, fs4, walkdir, ureq, tracing, tracing-subscriber. Test: tempfile.

**Modüller:**

| Modül | Sorumluluk |
| --- | --- |
| `layout` | `/srv/site` kökünden bütün yollar (`--root` ile değişir; testler geçici dizin kullanır) |
| `config` | `config/deployctl.toml` okuma ve doğrulama |
| `release` | `release.json` tipi, paket açma, doğrulama |
| `state` | `active.json` tipi, atomik yazma (geçici dosya + fsync + rename) |
| `lock` | `deploy.lock` üzerinde özel kilit; zaman aşımıyla bekleme |
| `assets` | Ortak varlık deposu, `assets-index.json`, 30 gün kuralı |
| `prune` | Eski sürüm klasörlerini temizleme |
| `system` | Dış dünya sınırı: `ServiceManager`, `Proxy`, `Http` trait'leri; gerçek ve sahte uygulamalar |
| `package` | CI/yerel tarafı: sürüm paketini oluşturma ve tutarlılık denetimi |

**Paket içeriği:**

```text
release-<sha>.tar.gz
  release.json
  site/            Astro dist (internal/ hariç)
  caddy/           site-routes.caddy, security-headers.caddy (İP-1.7)
  bundle/          (Faz 2A'dan itibaren)
  bin/             deployctl (her zaman), site-api (Faz 2B'den itibaren)
```

**Veri biçimleri:**

`release.json`:

```json
{
  "schemaVersion": 1,
  "commit": "<40 hex>",
  "builtAt": "2026-11-02T10:00:00Z",
  "siteMode": "prelaunch",
  "contractVersion": null,
  "components": { "site": true, "bundle": false, "api": false },
  "assets": ["_astro/index.Bx12ab.js"],
  "smoke": { "paths": ["/", "/projeler", "/hakkimda"], "searchQuery": null },
  "cspScriptHashes": []
}
```

`active.json`:

```json
{
  "schemaVersion": 1,
  "active": { "commit": "<sha>", "color": null, "activatedAt": "<zaman>" },
  "history": [
    { "commit": "<sha>", "color": null, "activatedAt": "<zaman>", "deactivatedAt": "<zaman>" }
  ]
}
```

`assets-index.json` (`/srv/site/assets-index.json`): varlık yolu → `lastUsedUntil`. Güncelleme kuralı her geçişte:

- Gelen sürümün varlıkları → `null` (kullanımda).
- Giden sürümün varlıklarından gelen sürümde **olmayanlar** → şimdiki zaman.
- Geri almada hedef sürümün varlıkları sürüm klasöründen yeniden eklenir (silinmişse) ve `null` olur.

`config/deployctl.toml`: `site_url`, `caddy_validate_cmd` (ör. `["sudo", "-n", "-u", "caddy", "caddy", "validate", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]`), `caddy_reload_cmd` (`["sudo", "-n", "systemctl", "reload", "caddy.service"]`), `service_cmd_prefix` (`["sudo", "-n", "systemctl"]`), `lock_timeout_secs` (900), `ready_timeout_secs` (60), `smoke_timeout_secs` (30), `asset_retention_days` (30), `keep_releases` (5).

**Komutlar (bu İP):**

- `deployctl package --site <dir> --caddy <dir> [--bundle <dir>] [--api-bin <file>] --deployctl-bin <file> --commit <40 hex> --mode <prelaunch|public> --out <dir>`: `release-<sha>.tar.gz` ve `.sha256` üretir. `site/internal/` pakete girmez; `assets` listesi `site/_astro/**`'dan çıkarılır. **Tutarlılık denetimi:** `site/index.html`'deki `release` meta değeri `--commit` ile aynı değilse; `--bundle` verildiyse `bundle/manifest.json` → `release.gitCommit` aynı değilse paket oluşturulmaz (D-02).
- `deployctl status`: etkin sürüm, geçmiş, kilit durumu, (ileride) çalışan renkler.
- `deployctl prune [--dry-run]`.

**Kurallar:**

- Varlık eklerken dosya zaten varsa blake3 ile karşılaştırılır; farklıysa işlem durur.
- Varlık temizliği: yalnız `lastUsedUntil` 30 günden eski olanlar silinir; indekste olmayan dosyalar **silinmez**, raporlanır. Boş veya okunamayan indeks hiçbir dosyanın silinmesine yol açmaz.
- Sürüm temizliği: etkin sürüm ve geçmişteki en son `keep_releases` sürüm silinmez.
- Kilit zaman aşımında açık hata: "Başka bir dağıtım sürüyor (kilit N sn'dir tutuluyor)".
- Bütün dosya yazımları atomik.

**Testler:** paket oluşturma/okuma; commit tutarsızlığında paket reddi; bozuk `release.json` reddi; atomik durum yazımı; iki iş parçacığıyla kilit yarışı; varlık ekleme (yeni, aynı, çakışan); indeks güncelleme kuralı (paylaşılan varlık zaman damgası almıyor); temizlik (29. gün silinmez, 31. gün silinir, etkin sürümünkü ve indekste olmayan asla); sürüm temizliği.

**Bitti sayılır:**

- [ ] `just check` Rust adımlarıyla birlikte geçiyor.

---

## İP-1.9 — deployctl: Caddy geçişi, duman testi, geri dönüş

**Amaç:** Statik sitenin kilit altında, test edilerek ve geri alınabilir biçimde devreye alınması. API'li mavi/yeşil adımlar İP-2B.6'da eklenir.

**Komutlar:**

- `deployctl deploy --package <file> --expect-active <sha|none> [--allow-older] [--dry-run]`
- `deployctl rollback --to <sha>` (= mevcut bir sürüm klasörünü `--allow-older` ile yeniden devreye almak)
- `deployctl verify` (uzlaştırma + duman testi; değişiklik yapmaz)

**`active.caddy` biçimi** (Caddy'nin ana dosyası site bloğu içinde bunu içe aktarır, İP-1.11):

```caddy
# deployctl tarafından üretilir — elle düzenlemeyin
import /srv/site/releases/<sha>/caddy/security-headers.caddy
import /srv/site/releases/<sha>/caddy/site-routes.caddy /srv/site/releases/<sha>/site /srv/site/assets 127.0.0.1:8080
```

API'siz sürümde API adresi `127.0.0.1:8080` kalır ve `/api` 502 döner.

**Akış (`deploy`):**

1. Kilidi al (zaman aşımıyla).
2. `active.json` oku. Etkin commit `--expect-active` ile aynı değilse dur: "Beklenen etkin sürüm X, bulunan Y; arada başka bir dağıtım olmuş."
3. Uzlaştırma: `active.caddy`, `active.json`'dan üretilecek içerikle aynı değilse dur ve uyar. Otomatik düzeltme yok. İstisna: `active.json` yoksa ve `active.caddy` ilk satırında `# deployctl:bootstrap` işareti taşıyorsa tutarlıdır.
4. Paketi doğrula (`.sha256`), `releases/<sha>.tmp` altına aç, `release.json`'ı doğrula, `releases/<sha>`'ya taşı. Klasör varsa ve içerik aynıysa yeniden kullan.
5. Paketin commit'i geçmişte varsa ve `--allow-older` yoksa dur.
6. Varlıkları ortak depoya ekle.
7. *(İP-2B.6: API'li sürümde boştaki rengi başlat ve doğrudan test et.)*
8. Yeni `active.json`'ı yaz; `active.caddy`'yi üret (öncekini `active.caddy.prev` olarak sakla); `caddy_validate_cmd`; geçerliyse `caddy_reload_cmd`.
9. Caddy üzerinden duman testi: `smoke.paths` her biri 200; `/` yanıtında `<meta name="release" content="<sha>">`; bir varlık yolu 200. İstekler alan adını 127.0.0.1'e çözerek (SNI korunarak) yapılır.
10. Başarılı: `assets-index.json` güncelle, *(İP-2B.6: eski rengi durdur)*, `prune`, kendi binary'sini `/srv/site/bin/deployctl`'a kopyala, kilidi bırak.
11. Başarısız (8. veya 9. adım): `active.json` ve `active.caddy` önceki hâline döner, reload, eski sürüm için duman testi tekrarlanır, hata ve geri dönüş raporlanır, çıkış ≠ 0. Geri dönüş aynı kilit içindedir; yalnız bu dağıtımın geçişini geri alır.

**Testler (sahte sistemle):** ilk dağıtım (`none`); `--expect-active` uyuşmazlığı → hiçbir dosya değişmez; geçmişteki commit `--allow-older` olmadan reddedilir, `rollback` ile kabul edilir; validate hatası → değişiklik yok; duman testi hatası → geri dönüş; yarıda kalmış dağıtım → sonraki çalıştırma durur; bootstrap işaretli ilk durum kabul edilir; iki eşzamanlı `deploy` seri çalışır ve ikincisi `expect-active` ile reddedilir; `--dry-run` hiçbir şeyi değiştirmez.

**Bitti sayılır:**

- [ ] Testler geçiyor; `deployctl --root <geçici> --dry-run deploy …` anlaşılır bir plan yazdırıyor.

---

## İP-1.10 — Sürüm paketi: `just release`

**Yapılacaklar:**

1. `just release mode="prelaunch"`: **tek çalıştırmada, tek commit'ten** bütün parçaları üretir:
   1. Çalışma ağacı kirliyse dur ("Önce commit'leyin").
   2. `SITE_RELEASE = git rev-parse HEAD` (40 hex).
   3. Web derlemesi (`SITE_MODE`, `SITE_RELEASE`, `SITE_URL`).
   4. *(Faz 2A: aynı `dist`'ten export → index.)*
   5. `cargo build --release` (deployctl; Faz 2B'den itibaren site-api).
   6. `deployctl package` → `out/release/`. Tutarlılık denetimi paketlemede yapılır (İP-1.8).
2. `just release-dry-run`: paketi geçici bir köke `--dry-run` ile dağıtır, planı gösterir.

**Bitti sayılır:**

- [ ] `out/release/release-<sha>.tar.gz` ve `.sha256` oluşuyor; pakette `site/internal/` yok, `caddy/` var; `release.json` doğru.
- [ ] Elle değiştirilmiş `release` meta değeriyle paketleme reddediliyor (test).

---

## İP-1.11 — Sunucu kurulum betikleri ve runbook

**Amaç:** Sunucuyu tekrarlanabilir biçimde hazırlamak. Hedef: Debian/Ubuntu LTS (A-01 farklıysa uyarlanır).

**Yapılacaklar:**

1. `deploy/bootstrap/bootstrap.sh` (idempotent; root ile; her adımı yazar; parametreler: alan adı, ACME e-postası):
   - `deploy` kullanıcısı (yalnız SSH anahtarı).
   - Dizinler: `/srv/site/{releases,assets/_astro,config,models,incoming,bin,run}`; sahip `deploy`; `releases` ve `assets` herkesçe okunabilir.
   - Caddy (resmî paket deposundan).
   - `/etc/caddy/Caddyfile`, alan adı dosyaya **doğrudan yazılarak** (ortam değişkeni yok).
   - İlk `active.caddy`: ilk satırı `# deployctl:bootstrap`, bakım sayfası gösteren geçici içerik.
   - `/etc/sudoers.d/site-deploy` (`visudo -c` ile doğrulanır): `deploy` yalnız `systemctl reload caddy.service` ve `caddy` kullanıcısı olarak `caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile` çalıştırabilir. (İP-2B.7 ve İP-4.1 servis komutlarını ekler.)
   - Güvenlik duvarı (ufw): 22, 80, 443.
   - `config/deployctl.toml` örnekten.
   - journald saklama: `/etc/systemd/journald.conf.d/site.conf` → `MaxRetentionSec=14day`, `SystemMaxUse=1G`.
2. `deploy/caddy/Caddyfile` (üretim; minimal): global seçenekler (`email`, `grace_period` **ayarlanmaz** — varsayılanı süren akışları kesmez; V2 §18.2), varsayılan logger için seviye `WARN` ve aşağıdaki filtreler; `<alan-adı>` site bloğu: erişim logu + `import /srv/site/active.caddy`.
3. **Log gizliliği (D-09; AGENTS.md §10):** erişim logu ve varsayılan logger aynı filtreyi kullanır. Taslak:

   ```caddy
   log {
   	output file {$CADDY_ACCESS_LOG:/var/log/caddy/access.log} {
   		roll_interval 24h
   		roll_keep_for 7d
   	}
   	format filter {
   		wrap json
   		fields {
   			request>uri query {
   				delete q
   			}
   			request>headers>Referer delete
   			request>headers>Cookie delete
   			request>remote_ip ip_mask {
   				ipv4 24
   				ipv6 48
   			}
   			request>client_ip ip_mask {
   				ipv4 24
   				ipv6 48
   			}
   		}
   	}
   }
   ```

   Sonuç: loglarda arama sorgusu, Referer ve tam IP yok; erişim logu en fazla ≈ 8 gün. Filtre ve seçenek adları kurulu Caddy sürümünde doğrulanır.
4. `deploy/config/deployctl.example.toml`.
5. `deploy/RUNBOOK.md` ilk sürüm: sunucu hazırlığı, ilk dağıtım, `deployctl status`, geri alma, takılmış kilit, sertifika sorunları; kopyalanabilir komutlar. SSH sertleştirme (parola ve root girişini kapatma) elle yapılacak adımlar olarak yazılır; betik SSH'ye dokunmaz.

**Bitti sayılır:**

- [ ] `bash -n deploy/bootstrap/bootstrap.sh` (Git Bash ile) temiz. shellcheck ve üretim Caddyfile doğrulaması İP-1.12'de CI'a eklenir.

---

## İP-1.12 — CI kalite iş akışı

**Önkoşul:** İP-1.7, İP-1.10, İP-1.11.

**Yapılacaklar:** `.github/workflows/ci.yml` (pull request ve `main` push'u):

| İş | Adımlar |
| --- | --- |
| `rust` | toolchain, önbellek, `just` kurulumu, Rust kontrolleri (`fmt`, `clippy -D warnings`, `test`, `cargo deny check`) |
| `web` | pnpm önbellek, `install --frozen-lockfile`, `astro check`, lint, format, vitest, `check-content`, `prelaunch` derlemesi, bağlantı ve kanonik URL denetimleri; `web/dist` artefakt |
| `e2e` | Caddy kurulumu, Playwright tarayıcıları, `just e2e`; başarısızlıkta rapor artefakt |
| `lighthouse` | `just lighthouse` |
| `deploy-config` | shellcheck (`deploy/**/*.sh`); `caddy validate` → `Caddyfile.local` ve sahte bir `active.caddy` ile üretim `Caddyfile` (`CADDY_ACCESS_LOG` geçici bir dosyaya yönlendirilir) |

Action'lar en az ana sürümle sabitlenir. Dal korumasında zorunlu tutulacak kontrol adları README'ye yazılır.

**Bitti sayılır:**

- [ ] Yerelde: iş akışı dosyası `actionlint` benzeri bir denetimden veya en azından YAML doğrulamasından geçiyor.
- [ ] **Kullanıcı kontrolü:** kullanıcı bir PR açtığında bütün işler yeşil; dal koruması açıldı.

---

## İP-1.13 — CD iş akışı ve ilk dağıtım

**Önkoşul:** İP-1.11, İP-1.12, İP-0.4 (sunucu), İP-0.5 (alan adı).

**Yapılacaklar:** `.github/workflows/deploy.yml` (`main` push):

1. `build-rust` işi: **sunucunun dağıtım sürümüyle aynı veya daha eski glibc'li** bir konteynerde (`container:`; ör. sunucu Debian 12 ise `rust:<sürüm>-bookworm`), mimari A-01'e göre. Yalnız Rust binary'lerini derler (`deployctl`; sonraki fazlarda `site-indexer`, `site-api`). Artefakt: binary'ler.
2. `build-web` işi (normal runner; Node, pnpm, just): web derlemesi (`SITE_MODE = vars.SITE_MODE`, `SITE_RELEASE = github.sha`, `SITE_URL = vars.SITE_URL`). Artefakt: `dist`.
3. `package` işi (normal runner): artefaktları indirir, binary'lere `chmod +x` uygular (artefaktlar çalıştırma iznini korumaz), *(Faz 2A: export'tan bundle)*, `deployctl package`. Artefakt: paket, `.sha256`, `deployctl`.
4. `deploy` işi:
   - `concurrency: { group: production-deploy, cancel-in-progress: false }`; `environment: production`.
   - `actions/checkout` `fetch-depth: 0`.
   - SSH ile `/srv/site/active.json` okunur → `ACTIVE` (yoksa `none`).
   - `ACTIVE` ≠ `none` ise `git merge-base --is-ancestor $ACTIVE $GITHUB_SHA`; değilse dur: "Canlıdaki commit bu commit'in atası değil."
   - scp ile paket, `.sha256` ve `deployctl` → `/srv/site/incoming/`; sunucuda `chmod +x /srv/site/incoming/deployctl-<sha>`.
   - `ssh … '/srv/site/incoming/deployctl-<sha> deploy --package … --expect-active $ACTIVE'`; sıfır olmayan çıkış işi düşürür.
5. Secret'lar: `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS`. Değişkenler: `SITE_URL`, `SITE_MODE`.
6. `deploy/RUNBOOK.md`'ye "GitHub ayarları" bölümü.

**Bitti sayılır (kullanıcı kontrolü; sunucu gerekir):**

- [ ] Bootstrap çalıştırıldı, secret'lar girildi, push sonrası site HTTPS ile açılıyor; HTML'de doğru `release` meta değeri ve noindex var.
- [ ] Erişim logunda arama sorgusu, Referer ve tam IP yok (`/ara?q=deneme` isteğinden sonra log satırı incelenir).
- [ ] Geri alma tatbikatı: A → B → `deployctl rollback --to A` → site A → yeniden B.
- [ ] Art arda iki push: ikinci push'un sürümü canlıda kalıyor.

---

## Faz 1 çıkış kapısı

**Kod kapısı** (Faz 2A ve Faz 3 için yeterli):

- [ ] İP-1.1 … İP-1.12 tamam; CI yeşil.
- [ ] Elle: site masaüstünde ve telefonda (yerel ağdan) okunuyor; JS kapalıyken gezinme çalışıyor; klavyeyle bütün bağlantılara ulaşılıyor.

**Sunucu kapısı** (sunucu hazır olunca):

- [ ] İP-1.13 tamam.

**Faz 2A'ya devir:** İçerik şemaları ve `content.ts` export için tek giriş noktasıdır. `deployctl package` `--bundle` parametresini ve commit tutarlılık denetimini şimdiden destekler.

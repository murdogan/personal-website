# Faz 2B — Arama API'si, Arama Adası ve Mavi/Yeşil Geçiş

**Amaç:** `site-api`'yi ayağa kaldırmak, aramayı sitede çalıştırmak ve API'li sürümleri kesintisiz devreye almak.
**Önkoşul:** Faz 2A çıkış kapısı.
**Referans:** V2 §4 (süreçler), §7.1 (sürüm farkı), §8 (Rust backend), §11 (HTTP sözleşmesi), §12.5 (arama adası), §16, §18.2.
**Kapsam dışı:** Asistan endpoint'leri (Faz 4; yapılandırmada yeri ayrılır), nihai görsel tasarım (Faz 3).

Rust odağı: Axum, Tokio, `Arc` ile paylaşılan durum, extractor'lar, middleware (Tower katmanları), async temelleri.

Komut kuralları İP-1 girişindeki gibidir: Rust komutları kökten `--manifest-path backend/Cargo.toml`, ortam değişkenleri `just` parametreleriyle.

---

## İP-2B.1 — `site-api` iskeleti

**Crate:** `backend/crates/api` (bin `site-api`). Bağımlılıklar: axum, tokio, tower, tower-http, figment, serde, serde_json, thiserror, anyhow, tracing, tracing-subscriber, metrics, metrics-exporter-prometheus, uuid, governor (İP-2B.2), contract, search.

**Modül düzeni:**

```text
api/src/
  main.rs          CLI, yapılandırma, telemetri, bundle yükleme, iki dinleyici, kontrollü kapanış
  config.rs        Config + validate()
  state.rs         AppState
  error.rs         ApiError → problem+json; extractor reddi eşlemesi
  client_ip.rs     güvenilir proxy kuralıyla istemci IP'si
  middleware.rs    sözleşme sürümü, X-Site-Release, Cache-Control, istek kimliği, istek logu
  metrics.rs
  routes/health.rs
  routes/search.rs     (İP-2B.2)
  routes/assistant.rs  (Faz 4)
```

**CLI:** `site-api --config <toml> --bundle <dir> [--listen <addr>] [--metrics-listen <addr>] [--log-format json|pretty]`. Sürüm kimliği bundle `manifest.json` → `release.gitCommit`'ten okunur (Astro'nun `SITE_RELEASE`'i ile aynı değer; İP-1.8 paketleme denetimi bunu garanti eder).

**Yapılandırma** (figment: TOML + `SITE_API__` önekli ortam değişkenleri):

```toml
[server]
environment = "production"                # production | development
listen = "127.0.0.1:8080"
metrics_listen = "127.0.0.1:9090"
public_origin = "https://example.com"     # Origin denetimi (Faz 4)
trusted_proxies = ["127.0.0.1", "::1"]
drain_timeout_secs = 90
body_limit_bytes = 16384

[search]
rate_per_10s = 20
default_limit = 8
max_limit = 20
max_query_chars = 200

[assistant]
mode = "off"            # off | fake | llama  (Faz 4)
```

Dosyalar:

| Dosya | Amaç | `environment` | `assistant.mode` |
| --- | --- | --- | --- |
| `config/site-api.example.toml` | Sunucuya kopyalanan şablon | `production` | `off` (Faz 4'te `llama`) |
| `config/site-api.dev.toml` | `just dev` | `development` | `off` (Faz 4'te `fake`) |
| `config/site-api.ci.toml` | CI e2e | `development` | `off` (Faz 4'te `fake`) |

`fake` modu yalnız `environment = "development"` iken kabul edilir; üretimde yapılandırma doğrulaması hata verir (ziyaretçiye sahte cevap gitmesin).

**Davranışlar:**

1. Açılış sırası: yapılandırma doğrula → bundle aç (`search::Engine::open`) → dinleyicileri bağla. Başarısızlıkta anlaşılır mesaj, çıkış ≠ 0.
2. Telemetri: `tracing` JSON (üretim) / okunabilir (geliştirme); `RUST_LOG`. Her istekte `x-request-id` (yoksa UUID üretilir, yanıta eklenir).
3. **İstek logu yalnız rota şablonunu yazar:** span alanları yöntem, `MatchedPath` (ör. `/api/search`), durum, süre, istek kimliği. URI, sorgu dizesi, başlıklar ve gövde loglanmaz. `TraceLayer::new_for_http()`'nin varsayılan span'ı **kullanılmaz** (URI'yi sorgu dizesiyle yazar); özel `make_span_with` yazılır. Eşleşmeyen yol için `MatchedPath` yerine `unmatched` yazılır.
4. `ApiError` → `application/problem+json` (AGENTS.md §6.4). `traceId` = istek kimliği. 429/503'te `Retry-After` başlığı ve `retryAfter` alanı.
5. Extractor reddi eşlemesi: geçersiz JSON / eksik alan → 400 `invalid_request`; yanlış `Content-Type` → 415 `invalid_request`; gövde sınırı aşımı → 413 `payload_too_large`. Bilinmeyen `/api/*` yolu → 404 `not_found`. Hepsi problem+json.
6. Middleware:
   - Her yanıtta `X-Site-Release: <commit>` ve `Cache-Control: no-store`.
   - `X-Contract-Version` varsa ve `MIN_SUPPORTED_CONTRACT`'tan küçükse 409 `client_outdated`. Başlık yoksa kabul edilir.
7. Sağlık: `GET /api/health/live` → `{ "status": "ok" }`; `GET /api/health/ready` → `{ "status": "ready", "release": "<commit>", "contractVersion": 1, "assistant": "disabled" }`.
8. Metrikler ayrı dinleyicide (`/metrics`): istek sayısı ve süre histogramı (rota şablonu, durum), `site_release_info{commit}` = 1.
9. Kontrollü kapanış: SIGTERM/Ctrl-C → yeni bağlantı kabul edilmez, süren istekler en fazla `drain_timeout_secs` beklenir, sonra çıkılır.

**Testler** (`tower::ServiceExt::oneshot`, fixture bundle; istemci adresi `MockConnectInfo` ile): sağlık yanıtları; problem+json biçimi; extractor reddi eşlemeleri; 409 `client_outdated`; `X-Site-Release` ve `Cache-Control`; bilinmeyen yol 404; `production` + `fake` yapılandırması reddediliyor; kapanışta süren isteğin tamamlanması (`tokio::time::pause`); log yakalayıcıyla bir arama isteğinin logunda sorgu metninin bulunmadığı.

**Bitti sayılır:**

- [ ] `cargo run --manifest-path backend/Cargo.toml -p api -- --config config/site-api.dev.toml --bundle out/bundle` çalışıyor; `curl.exe http://localhost:8080/api/health/ready` doğru sürümü döndürüyor.
- [ ] Testler ve clippy temiz.

---

## İP-2B.2 — Arama endpoint'i ve hız sınırı

**Endpoint:** `GET /api/search?q=<metin>&tags=<id,id>&limit=<n>`

1. Doğrulama `search::SearchRequest` kurallarıyla; hata → 400 `invalid_request` (alan bilgisiyle).
2. `Engine::search`; süre `search_duration_seconds` histogramına. Ölçülen p95 1 ms'yi aşarsa çağrı `spawn_blocking`'e alınır (rapor ölçümü belirtir).
3. Yanıt (`contract::api`, camelCase). Bölüm URL'si export'tan gelir; Rust URL kurmaz:

   ```json
   {
     "release": "a1b2c3…",
     "items": [{
       "id": "project/ornek-proje-1", "kind": "project",
       "title": "Örnek Proje 1", "summary": "…", "url": "/projeler/ornek-proje-1",
       "tags": [{ "id": "csharp", "label": "C#" }],
       "section": { "heading": "Çözüm", "url": "/projeler/ornek-proje-1#çözüm" },
       "snippet": [{ "text": "… ", "highlighted": false }, { "text": "öğrenci", "highlighted": true }]
     }],
     "matchedTags": [{ "id": "csharp", "label": "C#" }],
     "tookMs": 2
   }
   ```

4. Hız sınırı (`ratelimit.rs`): `governor` ile IP anahtarlı; 10 saniyede 20 istek. Aşımda 429 `rate_limited` + `Retry-After`. Anahtar `client_ip.rs`'den: karşı uç `trusted_proxies` içindeyse `X-Forwarded-For`'un **son** değeri, değilse karşı uç adresi. Sınırlayıcı belleği 60 sn'de bir temizlenir (`retain_recent`).

**Testler:** sonuçlar ve DTO biçimi; boş sorgu 400; 200 karakter sınırı; `limit` sınırları; aynı IP'den 21. istek 429; güvenilmeyen uçtan gelen `X-Forwarded-For` yok sayılıyor; güvenilir uçtan gelen farklı XFF değerleri ayrı sayılıyor; yanıtta puan alanı yok.

**Bitti sayılır:**

- [ ] `curl.exe "http://localhost:8080/api/search?q=ogrenci"` anlamlı sonuç döndürüyor.

---

## İP-2B.3 — TS tip üretimi ve API istemcisi

**Yapılacaklar:**

1. `contract::api` DTO'ları: `SearchResponse`, `SearchHitDto`, `SectionRefDto`, `SnippetFragment`, `TagDto`, `HealthReady`, `Problem`, `ErrorCode`. Süre ve sayaç alanları `u32` (ts-rs 64 bit tamsayıları `bigint` yapar; TS tarafında `number` istenir).
2. Üretim, `cargo test` sırasında değil, ayrı bir binary ile yapılır: `backend/crates/contract/src/bin/gen-types.rs`. ts-rs'nin programatik dışa aktarma işlevleriyle (`#[ts(export)]` **kullanılmaz**; her test çalıştırmasında dosya yazmasın) `web/src/generated/` altına yazar. ts-rs sabitleri dışa aktarmadığı için aynı binary `contract-version.ts`'i (`CONTRACT_VERSION`, `MIN_SUPPORTED_CONTRACT`) elle üretir.
3. `just gen-types`: binary + Prettier ile biçimlendirme. ESLint `generated/`'ı yok sayar.
4. CI: `just gen-types` sonrası `git diff --exit-code web/src/generated`.
5. `web/src/lib/api/`:
   - `client.ts`: `apiGet<T>(path, params, { signal })` → `{ ok: true, data: T } | { ok: false, problem: ApiProblem }`. Her istekte `X-Contract-Version`. Ağ hatası `problem.code = "network"` (istisna fırlatmaz; `AbortError` hariç).
   - `problem.ts`: problem+json ayrıştırma; saf veri.
   - `outdated.ts`: `client_outdated` gelince `site:client-outdated` olayı; `BaseLayout`'taki küçük bir bileşen "Site güncellendi — sayfayı yenile" bandını bir kez gösterir.

**Testler (Vitest):** başarı, problem+json, ağ hatası, iptal, `client_outdated` olayı.

**Bitti sayılır:**

- [ ] Bir DTO alanı değişince `just gen-types` sonrası fark oluşuyor ve TS derlemesi uyumsuz kullanımı yakalıyor (yerelde gösterilir, geri alınır).

---

## İP-2B.4 — Geliştirme ortamı: `just dev`

**Yapılacaklar:**

1. `just dev`: önce `just index` (API çalışmıyorken; Windows'ta bundle kilitlenmesin), sonra `concurrently` ile:
   - `pnpm --filter web dev` (4321),
   - `cargo run --manifest-path backend/Cargo.toml -p api -- --config config/site-api.dev.toml --bundle out/bundle` (8080).
   Astro dev `/api`'yi 8080'e iletir (İP-1.2).
2. `just dev-web`: yalnız Astro (arama "kullanılamıyor" durumuna düşer).
3. README "Geliştirme": içerik değişince aramayı güncellemek için `just dev`'i durdurup yeniden başlatmak (veya API'yi durdurup `just index`); port çakışmasında ne yapılacağı.

**Bitti sayılır:**

- [ ] Temiz bir klonda `just setup`, ardından `just dev` Windows'ta çalışıyor; tarayıcıda `http://localhost:4321/api/health/ready` yanıt veriyor.

---

## İP-2B.5 — Arama adası ve `/ara` sayfası

**Referans:** V2 §12.5; WAI-ARIA APG combobox deseni.

**Bileşenler** (`web/src/islands/search/`):

1. `SearchBox.tsx` — üst bölümdeki arama alanı (`client:idle`; `idPrefix` prop'u — Faz 3'te ikinci kopya için):
   - SSR çıktısı İP-1.4'teki formla aynı işaretlemedir; JS yokken form `/ara?q=…`'ya gider.
   - Combobox: `input role="combobox"`, `aria-expanded`, `aria-controls`, `aria-activedescendant`; liste `role="listbox"`, öğeler `role="option"`.
   - En az 2 karakter; 150 ms gecikme; `AbortController` ile önceki istek iptal; artan sıra numarası; geç gelen eski yanıt yok sayılır.
   - En fazla 8 sonuç: başlık, (varsa) bölüm başlığı, vurgulu parça (`<mark>`), etiketler. Metin React metin düğümü olarak basılır; `dangerouslySetInnerHTML` yok.
   - Klavye: ↑/↓ seçim, `Enter` seçili sonuca (seçim yoksa `/ara?q=`), `Esc` listeyi kapatır, ikinci `Esc` alanı temizler. `/` kısayolu alana odaklanır (başka bir giriş alanındayken çalışmaz).
   - `aria-live="polite"` durum metni: "5 sonuç", "Sonuç bulunamadı", "Arama şu an kullanılamıyor".
   - Hata/ağ yok: "Arama şu an kullanılamıyor" + "Projelerde etiketle gez" (`/projeler`).
   - Sonuç yok: "Sonuç bulunamadı" + kullanılan etiketlerden 5 öneri (Astro prop verir).
2. `SearchPage.tsx` — `/ara` (`client:load`): URL'den `q` ve `tags`; en fazla 20 sonuç; etiket çipleri (Must filtresi); değişiklikte `history.replaceState`. `<noscript>` içeriği kalır.
3. Durum mantığı saf bir reducer'da (`searchReducer.ts`).
4. Stil: İP-1.5 token'larıyla sade; nihai görünüm İP-3.2/İP-3.5'te.

**Testler:**

- Vitest: reducer; sıra numarası koruması; gecikme.
- Playwright (`page.route` ile taklit): klavye akışı; `Esc`; `/` kısayolu; hata ve boş durum; durum metni; axe.

**Bitti sayılır:**

- [ ] `just dev` ile gerçek bundle'da arama çalışıyor (elle: Türkçe karaktersiz, teknoloji adı, yazım hatası).
- [ ] JS kapalıyken form `/ara`'ya gidiyor, noscript içeriği görünüyor.
- [ ] Lighthouse JS bütçesi (≤ 100 KB) geçiyor.

---

## İP-2B.6 — Sürüm hattına API ve mavi/yeşil geçiş

**Referans:** V2 §18.2 adım 3–7; İP-1.9 akışı.

**Sürüm hattı (D-02):**

1. `just release` (İP-1.10 adım 5): `cargo build --release` artık `site-api`'yi de derler; paketleme `--api-bin` ile `bin/site-api` ekler; `release.json` → `components.api = true`, `contractVersion = CONTRACT_VERSION`.
2. `smoke.searchQuery`: bundle'daki ilk öne çıkan öğenin başlığının ilk kelimesi (sonuç döndüğü bilinen sorgu).
3. Deploy iş akışı (İP-1.13): `build-rust` işi `site-api`'yi derler; `package` işi `--api-bin` geçirir.

**Renk düzeni:**

- `blue` (API 8080, metrik 9090), `green` (8082, 9092). `active.json` → `active.color`.
- Her renk için sabit yol: `/srv/site/run/<renk>/site-api` ve `/srv/site/run/<renk>/bundle`, ilgili sürüm klasörüne **symlink**'tir (systemd, `ExecStart`'ın ilk argümanında değişkene izin vermez; İP-2B.7). Symlink'ler geçici ad + `rename` ile atomik değiştirilir.
- `/srv/site/config/site-api-<renk>.env` yalnız `LISTEN` ve `METRICS_LISTEN` içerir; bootstrap yazar, deployctl dokunmaz.

**deployctl akışı eklemeleri** (İP-1.9 adım 7, `components.api = true` iken):

1. Boştaki renk: etkin renk yoksa `blue`, varsa diğeri.
2. O renk çalışıyorsa (yarıda kalmış dağıtımdan) önce durdurulur.
3. `run/<renk>/` symlink'leri yeni sürüme çevrilir.
4. `systemctl start site-api@<renk>.service`.
5. Doğrudan porttan, `ready_timeout_secs` içinde: `/api/health/ready` 200 ve `release == commit`; `/api/search?q=<searchQuery>` 200 ve en az 1 sonuç. Başarısızsa yeni renk durdurulur, Caddy'ye dokunulmaz, hata raporlanır.
6. `active.caddy`'deki API adresi yeni rengin portu olur; Caddy üzerinden duman testine `/api/health/ready` (sürüm eşleşmesi) eklenir.
7. Başarılı: eski renk `systemctl stop` ile durdurulur (birim `TimeoutStopSec=120`; API 90 sn içinde drene olur); bu bitmeden kilit bırakılmaz.
8. Geçiş sonrası hata: `active.*` geri alınır, Caddy yeniden yüklenir (eski renk hâlâ çalışıyor), yeni renk durdurulur.
9. Uzlaştırma ek kuralı: `active.color` çalışıyor olmalı; diğer renk çalışıyorsa dur ve uyar.
10. API'siz bir sürüme geri dönüş reddedilir: "Hedef sürümde API yok."
11. `deployctl status` çalışan renkleri ve symlink hedeflerini gösterir.

**Testler (sahte sistem):** mavi → yeşil → mavi; ready başarısız → Caddy'ye dokunulmuyor, yeni renk durduruluyor; geçiş sonrası duman hatası → geri dönüş, eski renk çalışıyor; boştaki renk zaten çalışıyor → önce durduruluyor; iki renk birden çalışıyor → uzlaştırma durduruyor; API'siz sürüme dönüş reddi; symlink değişimi atomik (geçici ad + rename).

**Bitti sayılır:**

- [ ] Testler geçiyor; `just release` paketinde `bin/site-api` var ve `release.json` `components.api = true`.

---

## İP-2B.7 — Sunucu: systemd şablonu, sudoers, bootstrap güncellemesi

**Yapılacaklar:**

1. `deploy/systemd/site-api@.service`:

   ```ini
   [Unit]
   Description=site-api (%i)
   After=network.target

   [Service]
   User=siteapi
   Group=siteapi
   EnvironmentFile=/srv/site/config/site-api-%i.env
   EnvironmentFile=-/srv/site/config/secrets.env
   ExecStart=/srv/site/run/%i/site-api --config /srv/site/config/site-api.toml --bundle /srv/site/run/%i/bundle --listen ${LISTEN} --metrics-listen ${METRICS_LISTEN} --log-format json
   KillSignal=SIGTERM
   TimeoutStopSec=120
   Restart=on-failure
   RestartSec=2
   NoNewPrivileges=yes
   ProtectSystem=strict
   ProtectHome=yes
   PrivateTmp=yes
   MemoryMax=512M

   [Install]
   WantedBy=multi-user.target
   ```

   Çalıştırılabilir dosya yolu sabittir; değişkenler yalnız argüman konumlarındadır.
2. `bootstrap.sh` eklemeleri: `siteapi` kullanıcısı (giriş kabuğu yok); `run/blue`, `run/green` dizinleri (sahip `deploy`); `site-api-blue.env` (`LISTEN=127.0.0.1:8080`, `METRICS_LISTEN=127.0.0.1:9090`) ve `site-api-green.env` (`8082`/`9092`); `config/site-api.toml` örnekten; `secrets.env` (boş, izin 640, sahip `root`, grup `siteapi`); birim kurulumu ve `daemon-reload`.
3. sudoers eklemesi (`visudo -c`): `deploy` için yalnız `systemctl start|stop|reset-failed site-api@blue.service` ve `site-api@green.service`. `is-active` sudo gerektirmez.
4. RUNBOOK: API logları (`journalctl -u site-api@blue`), metrikler (`curl 127.0.0.1:9090/metrics`), elle renk değiştirmenin neden yapılmayacağı (her zaman `deployctl`).

**Bitti sayılır:**

- [ ] CI'da `systemd-analyze verify` (birim dosyası için) ve shellcheck temiz.
- [ ] **Kullanıcı kontrolü (sunucu):** bootstrap güncellemesi uygulandı; API'li ilk sürüm dağıtıldı; ikinci dağıtımda renk değişti; dağıtım sırasında Caddy üzerinden `/api/health/ready`'yi sürekli sorgulayan bir döngüde 200 dışı yanıt görülmedi.

---

## İP-2B.8 — Uçtan uca testler ve CI

**Yapılacaklar:**

1. CI `e2e` işi: web derle + export + bundle → `site-api` (`config/site-api.ci.toml`, `--bundle out/bundle`) arka planda → Caddy (`Caddyfile.local`, `SITE_API=127.0.0.1:8080`) → Playwright.
2. Gerçek API ile testler (`web/tests/e2e/search.spec.ts`): üst arama alanı ile sonuç bulma ve sonuca gitme; `/ara?q=`; etiket filtresi; API durdurulunca "kullanılamıyor"; konsolda hata yok.
3. Lighthouse bütçesi arama adasıyla yeniden doğrulanır.

**Bitti sayılır:**

- [ ] `just e2e` yerelde gerçek API ile geçiyor.
- [ ] **Kullanıcı kontrolü:** CI yeşil.

---

## Faz 2B çıkış kapısı

**Kod kapısı** (Faz 3'ün 3.2/3.5 İP'leri ve Faz 4'ün kod İP'leri için yeterli):

- [ ] İP-2B.1 … İP-2B.6 ve İP-2B.8 tamam; arama yerelde çalışıyor; API kapalıyken sayfalar ve form çalışıyor (D-04).
- [ ] Sayfanın `release` meta değeri ile API'nin `X-Site-Release` başlığı aynı (D-03; `just release` paketinden kurulan yerel ortamda).
- [ ] Tip üretimi farkı ve sözleşme denetimi çalışıyor.
- [ ] Elle: klavye ve ekran okuyucuyla (Narrator veya NVDA) arama kullanıldı.

**Sunucu kapısı** (Faz 4'ün sunucu İP'leri için):

- [ ] İP-2B.7 tamam; mavi/yeşil geçiş sunucuda en az iki kez sorunsuz yapıldı.

**Faz 3 ve 4'e devir:** API iskeleti, hata modeli, hız sınırı, istemci IP kuralı, `SearchBox` (`idPrefix`) ve mavi/yeşil geçiş hazır. Asistan rotaları `routes/assistant.rs`'e eklenecek.

# AGENTS.md — Kişisel Web Sitesi

Bu dosya, bu depoda çalışan AI kodlama ajanları (Codex vb.) ve insan geliştirici için **bağlayıcı** kurallardır. Her oturumda önce bu dosyayı, sonra size verilen iş paketinin (İP) bulunduğu faz belgesini okuyun.

## 1. Proje özeti

Kişisel portföy sitesi: tanıtım, projeler, üstte site içi arama, sağda yerel bir dil modeliyle çalışan site asistanı, çok dilli giriş animasyonu ve portre arkasında etkileşimli akış.

- İçerik git'te Markdown/MDX (`content/`). Yönetim paneli ve veritabanı yok.
- Frontend: Astro statik çıktı + React adaları (`web/`).
- Backend: Rust (`backend/`). `site-indexer` derleme zamanında arama/asistan paketini (bundle) üretir; `site-api` arama ve asistan API'sini sunar; `deployctl` sunucuda sürüm geçişini yönetir.
- Model: llama.cpp `llama-server`, ayrı süreç, yalnız localhost.
- Sunucu: Caddy + systemd. Sürüm geçişi mavi/yeşil API + tek `caddy reload`.

## 2. Belge haritası ve öncelik

| Belge | Ne için |
| --- | --- |
| `AGENTS.md` (bu dosya) | Kalıcı kurallar, sabitler, kod standartları |
| `fazlar/README.md` | Faz sırası, iş paketi listesi, ilerleme tablosu, istem şablonu |
| `fazlar/FAZ-*.md` | İş paketlerinin ayrıntılı tanımı ve bitti ölçütleri |
| `fazlar/SORULAR.md` | Açık kararlar ve ajanın sorduğu sorular |
| `TEKNIK_MIMARI_V2.md` | Mimari, değişmezler (D-01…D-12), gerekçeler |
| `TASARIM.md` | Görsel yön ve hareket |
| `PROJE_KAPSAMI.md` | Ürün kapsamı |
| `TEKNIK_MIMARI.md` | **v1, yalnız tarihsel.** Uygulamada kullanılmaz. |

Öncelik: `AGENTS.md` > ilgili `FAZ-*.md` > `TEKNIK_MIMARI_V2.md` > `TASARIM.md` > `PROJE_KAPSAMI.md`. Belgeler arasında çelişki görürseniz kendiniz çözmeyin: `fazlar/SORULAR.md`'ye yazın ve kullanıcıya sorun.

## 3. Çalışma biçimi: iş paketi (İP)

1. Bir oturumda **yalnız istenen İP** uygulanır. Başka İP'lerin işi yapılmaz. "Bu arada şunu da düzelteyim" yapılmaz; gördüğünüz sorunları raporda belirtin.
2. Başlamadan önce okuyun: İP metni, İP'nin referans verdiği V2 bölümleri ve İP'nin dokunacağı mevcut dosyalar.
3. Önkoşul İP'ler tamamlanmamışsa başlamayın; raporla bildirin.
4. Uygulayın, İP'nin istediği testleri yazın, `just check` çalıştırın. İP ek doğrulama istiyorsa (`just e2e`, `just lighthouse` vb.) onları da çalıştırın.
5. `fazlar/README.md`'deki ilerleme tablosunda İP'nin durumunu güncelleyin.
6. Değişiklikleri tek commit olarak hazırlayın: `[İP-1.3] İçerik şemaları ve örnek içerik`. **Push yapmayın**; kullanıcı inceleyip push eder.
7. Raporu aşağıdaki biçimde verin.

Belirsizlik kuralı: Kararı geri alınabilir ve küçükse en güvenli varsayımı yapın, kodda `// VARSAYIM:` yorumuyla işaretleyin ve raporda belirtin. Karar ürün, tasarım, güvenlik veya veri biçimiyle ilgiliyse durun, soruyu `fazlar/SORULAR.md`'ye ekleyin ve kullanıcıya sorun.

Rapor biçimi:

```text
## İP-x.y raporu
Durum: Tamamlandı | Kısmen | Engellendi
Yapılanlar: (madde madde)
Değişen/eklenen dosyalar: (liste)
Doğrulamalar: (komut → sonuç; başarısızsa çıktının ilgili kısmı)
Bitti ölçütleri: (her madde ✓ veya ✗ + açıklama)
Varsayımlar ve sapmalar: (gerekçesiyle)
Açık sorular: (SORULAR.md'ye eklenenler)
Kullanıcının elle kontrol etmesi gerekenler: (varsa)
```

## 4. Değişmezler (özet)

Ayrıntı ve gerekçe: `TEKNIK_MIMARI_V2.md` §3. Bunları bozan bir değişiklik yapmayın; gerekiyorsa durup sorun.

- **D-01** Sunucuda tek kopyası bulunan kalıcı veri yok.
- **D-02** Sürüm = aynı commit'ten site + bundle + binary; parçalar ayrı güncellenmez.
- **D-03** Geçişten sonraki her yeni istek sayfa, arama ve asistanda aynı sürümü görür; eski sekmeler zararsız kılınır.
- **D-04** Public sayfalar JS, API ve model olmadan okunabilir.
- **D-05** Modelin araç, ağ, dosya veya yazma yetkisi yok.
- **D-06** Kaynak bağlantıları sunucunun seçtiği kaynak listesinden üretilir, modelin metninden asla.
- **D-07** Her pahalı kaynak sınırlı; sınır aşımı açık bir yanıttır.
- **D-08** Sözleşmenin sahibi üreten taraftır; tüketen CI'da doğrular.
- **D-09** Ziyaretçi soruları ve kimliği kalıcı saklanmaz.
- **D-10** Geri alma basittir ve denenir.
- **D-11** Dağıtım/geri alma tek kilit altında; otomatik süreç daha eski commit'e geçemez.
- **D-12** Yayımlanmış sayfanın hash'li varlıkları, sürümü devreden çıktıktan sonra en az 30 gün kalır.

## 5. Sahiplik ve sözleşmeler

- İçeriği ayrıştırmak, render etmek ve URL üretmek **yalnız Astro'nun** işidir. Rust Markdown render etmez ve URL kurmaz; bölüm bağlantıları dahil bütün URL'ler export'tan (`items[].url`, `sections[].url`) gelir.
- Arama sıralaması, bağlam seçimi ve model erişimi **yalnız Rust'ın** işidir. Frontend sonuçları yeniden sıralamaz veya süzmez.
- **Content export v1**: Astro üretir (`web/src/lib/export/`), Rust `contract::export` ile `deny_unknown_fields` kullanarak okur.
- **API DTO'ları ve SSE olayları**: `backend/crates/contract` içinde tanımlanır; `ts-rs` ile `web/src/generated/` altına üretilir.
- Üretilmiş dosyalar elle düzenlenmez: `web/src/generated/**`, `out/**`, `Cargo.lock` ve `pnpm-lock.yaml` (yalnız paket yöneticisi değiştirir).

## 6. Ortak sabitler

Bu adları değiştirmeyin; değiştirmek gerekiyorsa sorun.

### 6.1 Dizinler

```text
content/                     içerik (profile.md, projects/<slug>/index.mdx, tags.yaml)
web/                         Astro uygulaması
web/src/islands/             search/, assistant/, hero/, intro/
web/src/generated/           ts-rs çıktısı (elle düzenlenmez)
backend/crates/contract      export v1, API DTO, SSE olayları, CONTRACT_VERSION
backend/crates/search        Türkçe analiz, indeks, sorgu, bundle modeli
backend/crates/assistant     bağlam kartı, planlayıcı, prompt, model istemcisi, atıf
backend/crates/indexer       bin: site-indexer
backend/crates/api           bin: site-api
backend/crates/deployctl     bin: deployctl
backend/crates/eval          bin: site-eval
backend/fixtures/            test fixture'ları (export-v1.json vb.)
config/                      site-api yapılandırma dosyaları (dev ve örnek)
deploy/                      caddy/, systemd/, bootstrap/, llama/, tests/, RUNBOOK.md, model.toml
eval/                        search.toml, assistant.toml, reports/
docs/adr/                    karar kayıtları
out/                         derleme çıktıları (git'e girmez): export/, bundle/, dev-release/, release/
```

### 6.2 Portlar ve adresler

| Bileşen | Adres |
| --- | --- |
| Astro dev | `http://localhost:4321` |
| site-api mavi / dev | `127.0.0.1:8080`, metrik `127.0.0.1:9090` |
| site-api yeşil | `127.0.0.1:8082`, metrik `127.0.0.1:9092` |
| llama-server | `127.0.0.1:8081` |
| CI/yerel Caddy testi | `http://localhost:8088` |

### 6.3 Ortam değişkenleri (derleme)

| Değişken | Anlamı | Varsayılan |
| --- | --- | --- |
| `SITE_URL` | Kanonik site adresi | `http://localhost:4321` |
| `SITE_RELEASE` | Commit SHA | `dev` |
| `SITE_MODE` | `prelaunch` (noindex, yer tutucuya izin) veya `public` | `prelaunch` |

### 6.4 HTTP

- İstek başlığı: `X-Contract-Version: <n>` (istemci her API isteğinde gönderir).
- Yanıt başlığı: `X-Site-Release: <commit>` (API her yanıtta gönderir).
- HTML: `<meta name="release" content="<commit>">`.
- Hata biçimi: `application/problem+json`. Alanlar: `type`, `title`, `code`, `traceId`, isteğe bağlı `retryAfter`. Kodlar: `invalid_request` 400 (yanlış `Content-Type` için 415), `forbidden_origin` 403, `not_found` 404, `client_outdated` 409, `payload_too_large` 413 (gövde sınırı), `message_too_long` 413 (asistan mesajı), `rate_limited` 429, `assistant_busy` 503, `assistant_offline` 503, `internal` 500.
- SSE olayları: `meta`, `sources`, `delta`, `done`, `error`. Her akış tam olarak bir terminal olayla (`done` veya `error`) biter.

### 6.5 Sunucu yerleşimi

```text
/srv/site/releases/<commit>/{site/, caddy/, bundle/, bin/, release.json}
/srv/site/assets/_astro/       ortak varlık deposu
/srv/site/assets-index.json    varlık → lastUsedUntil (D-12 temizliği)
/srv/site/active.json          etkin sürüm durumu
/srv/site/active.caddy         deployctl üretir (Caddyfile site bloğu bunu içe aktarır)
/srv/site/deploy.lock
/srv/site/run/<renk>/          site-api ve bundle symlink'leri (blue | green)
/srv/site/config/              deployctl.toml, site-api.toml, site-api-blue.env, site-api-green.env,
                               secrets.env (LLAMA_API_KEY, ALERT_WEBHOOK_URL), llama.env, llama-api-key,
                               monitor.toml
/srv/site/models/
/srv/site/monitor-state.json
/srv/site/incoming/            CI'nın yüklediği paketler
/srv/site/bin/deployctl        son başarılı dağıtımın deployctl kopyası
```

## 7. Komutlar

`justfile` İP-1.1'de oluşturulur. Recipe'ler Windows (PowerShell) ve Linux'ta çalışmalıdır; karmaşık mantık shell'de değil Rust veya Node (TypeScript) betiklerinde yazılır.

Komut kuralları:

- Rust komutları depo kökünden `--manifest-path backend/Cargo.toml` ile çalışır; göreli yollar (`config/`, `out/`, `content/`) depo köküne göredir.
- Ortam değişkeni gereken recipe'ler `just`'ın `$` önekli parametreleriyle yazılır (`build $SITE_MODE="prelaunch":`). `VAR=x komut` ve `a && b` biçimleri Windows PowerShell 5.1'de çalışmaz; belgelerde ve recipe'lerde kullanılmaz.
- Belgelerdeki HTTP örnekleri `curl.exe` ile; JSON gövdesi dosyadan (`--data-binary @dosya.json`).

| Komut | İş |
| --- | --- |
| `just setup` | Bağımlılıkları kur (`pnpm install`, `cargo fetch`) |
| `just dev` | Bundle'ı kurar, sonra Astro dev + site-api (geliştirme yapılandırması) birlikte |
| `just dev-web` | Yalnız Astro dev |
| `just dev-model` | site-api'yi yerel llama-server'a bağlı başlatır (Faz 4) |
| `just check` | Format kontrolü, lint, tip kontrolü, birim testler, içerik denetimi (`prelaunch`). **Her İP sonunda çalışır.** |
| `just check-public` | İçerik denetimi ve derleme `public` kipinde (yer tutucu varsa başarısız) |
| `just fmt` | Biçimlendir |
| `just test` | Rust + web birim testleri |
| `just build` | Web derlemesi |
| `just export` | Content export → `out/export/content.v1.json` |
| `just index` | Export + bundle → `out/bundle/` |
| `just gen-types` | ts-rs tiplerini üret |
| `just e2e` | Playwright (Caddy üzerinden) |
| `just lighthouse` | Lighthouse CI bütçeleri |
| `just eval` | Arama ve geri getirme değerlendirmesi |
| `just release` | Tek commit'ten site + bundle + binary'ler → sürüm paketi (`out/release/`); kirli ağaçta durur |

## 8. Kod kuralları

### 8.1 Genel

- Kod tanımlayıcıları İngilizce. Kod yorumları ve belgeler Türkçe. Ziyaretçiye görünen tüm metinler Türkçe. Log mesajları İngilizce ve yapılandırılmış.
- Küçük, okunabilir fonksiyonlar. Gereksiz soyutlama yok: genel repository, event bus, DI çatısı, servis bulucu kurulmaz.
- Her davranış değişikliği test ile gelir. Testler uygulamayı satır satır tekrar etmez; iş kuralını veya hata senaryosunu doğrular.
- Gizli değer (anahtar, parola) repoya, loga veya frontend paketine girmez.

### 8.2 Rust

- Edition 2024; toolchain `rust-toolchain.toml` ile sabit.
- Workspace genelinde `unsafe_code = "forbid"`; `clippy::unwrap_used`, `clippy::expect_used` ve `clippy::panic` reddedilir (testlerde serbest, `clippy.toml`).
- Kütüphane crate'lerinde `thiserror` ile tipli hata enum'ları; binary `main` ve CLI katmanında `anyhow`.
- Kimlikler newtype (`ItemId`, `TagId`, `ChunkId`, `SourceLabel`).
- `dyn` async trait yerine somut tip veya enum ile dağıtım (`ModelClient` enum'u gibi).
- Sahiplik hatalarını gerekçesiz `clone()`, `Arc<Mutex<_>>`, `'static` veya `Box::leak` ile susturmayın; nedenini yorumla açıklayın.
- Kullanıcı girdisi asla Tantivy `QueryParser`'a verilmez; sorgular programatik kurulur.
- Async kodda bloklayan iş (`std::fs` büyük okuma, ağır CPU) `spawn_blocking` ile ayrılır.
- `cargo fmt`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, `cargo deny check` temiz olmalı.

### 8.3 TypeScript, Astro ve React

- TypeScript `strict` (Astro `strictest` şablonu). `any` yasak; gerekiyorsa `unknown` + daraltma.
- Astro bileşenleri statik içerik için; React yalnız adalarda (`web/src/islands/`).
- Ada yükleme yönergeleri faz belgelerinde belirtilir (`client:idle`, `client:visible` vb.). Varsayılan olarak ada eklenmez.
- Fare/kaydırma gibi sık olaylar React state'ine yazılmaz.
- Erişilebilirlik: anlamlı HTML, klavye ile kullanım, görünür odak, `alt` metni, ARIA yalnız gerektiğinde ve APG desenlerine uygun.
- CSS: token'lar `web/src/styles/tokens.css`'te custom property olarak; bileşen stilleri Astro kapsamlı stil. CSS çatısı (Tailwind vb.) eklenmez.
- `prefers-reduced-motion` her animasyonda dikkate alınır.
- ESLint + Prettier temiz; `astro check` hatasız.
- Ortam değişkenleri tek noktadan okunur: Astro kodu `web/src/lib/site.ts`, Node betikleri (`web/scripts/`) `web/scripts/lib/env.ts`.
- Sayfa yolları `web/src/lib/paths.ts` → `pagePath()` ile karşılaştırılır (`build.format: "file"` derlemede `.html` ekler).
- Sayfa betikleri `ClientRouter` yumuşak gezinmesinde yeniden çalışmaz: başlatma `astro:page-load`'da, temizlik `astro:before-swap`'ta yapılır.

## 9. Bağımlılık politikası

Aşağıdaki paketler önceden onaylıdır. Listede olmayan bir bağımlılık eklemeden önce durun ve gerekçeyle sorun. Sürümler, eklendiği gün güncel kararlı sürüm olarak seçilir ve lock dosyasıyla sabitlenir.

**Rust:** serde, serde_json, toml, thiserror, anyhow, clap (derive), tracing, tracing-subscriber, tokio, axum, tower, tower-http, tantivy, unicode-normalization, blake3, sha2, base64, ts-rs, reqwest (rustls, stream, json; varsayılan özellikler kapalı), async-stream, futures-util, governor, figment, time, url, regex, tar, flate2, fs4, walkdir, ureq, metrics, metrics-exporter-prometheus, uuid. Yalnız test: tempfile, insta, wiremock, proptest.

**Web:** astro, @astrojs/react, @astrojs/mdx, @astrojs/sitemap, @astrojs/check, typescript, react, react-dom, @types/react, @types/react-dom, @types/node, sharp, yaml, gsap, eventsource-parser, react-markdown, unified, remark-parse, remark-mdx, remark-gfm, mdast-util-to-string, unist-util-visit, @fontsource-variable/inter, tsx, concurrently. Yalnız geliştirme/test: vitest, @playwright/test, @axe-core/playwright, @lhci/cli, eslint, typescript-eslint, eslint-plugin-astro, eslint-plugin-react-hooks, eslint-plugin-jsx-a11y, prettier, prettier-plugin-astro.

**Araçlar:** just, cargo-deny, Caddy, shellcheck (CI).

**Lisanslar (cargo-deny):** MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC, Unicode-3.0, Zlib, MPL-2.0, CC0-1.0, CDLA-Permissive-2.0 (rustls kök sertifikaları). Listeye ekleme kullanıcı onayı ister.

## 10. Yasaklar

- Veritabanı, ORM, mesaj kuyruğu, yönetim paneli, kullanıcı hesabı, çerez tabanlı oturum eklemek.
- Ziyaretçi sorularını, cevapları veya tam IP adreslerini kalıcı yazmak (log dahil). Arama sorgu metnini loglamak. Bu, Caddy logları için de geçerlidir: sorgu dizesindeki `q`, `Referer` ve maskelenmemiş `remote_ip`/`client_ip` loga girmez (İP-1.11). site-api istek logu yalnız rota şablonunu yazar, URI'yi yazmaz.
- Modele araç/fonksiyon çağırma, ağ veya dosya erişimi vermek.
- Model metnindeki herhangi bir bağlantıyı (site içi yollar dahil) tıklanabilir yapmak. Tıklanabilir olanlar yalnız sunucunun `sources` listesine dayanan `[Kn]` numaraları ve kaynak kartlarıdır.
- Üretim yapılandırmasında `assistant.mode = "fake"` kullanmak (yalnız `environment = "development"`).
- Caddy'de `flush_interval -1` kullanmak (istemci kopunca arka uç iptalini kapatır).
- Gerçek içerik varmış gibi tanıtım metni, unvan, başarı veya proje bilgisi uydurmak. Yer tutucular `placeholder: true` ile işaretlenir.
- AI ile üretilmiş örnek portreyi üretim içeriği olarak kullanmak.
- Testi geçirmek için testi zayıflatmak, atlamak (`skip`, `#[ignore]`) veya bütçe eşiklerini gevşetmek. Gerekirse sorun.
- `--no-verify`, lint'i kapatma yorumları (`eslint-disable`, `#[allow(...)]`) gerekçesiz kullanmak. Gerekiyorsa satır içinde gerekçe yazın.
- `git push`, sürüm etiketi oluşturmak veya sunucuya bağlanmak (kullanıcı açıkça istemedikçe).

## 11. İçerik ve yer tutucu politikası

- Gerçek içerik gelene kadar `content/` altındaki örnekler `placeholder: true` taşır ve metinleri açıkça yer tutucudur ("Ad Soyad", "Örnek Proje 1").
- `SITE_MODE=public` derlemesi, herhangi bir `placeholder: true` içerik varsa başarısız olur.
- Test fixture'ları (`backend/fixtures/`, `web/tests/fixtures/`) gerçek içerikten bağımsızdır; içerik değişince testler kırılmaz.

## 12. Güvenlik ve gizlilik kontrol listesi (her İP)

- Yeni endpoint varsa: girdi doğrulama, boyut sınırı, hız sınırı, hata biçimi, log'da içerik yok.
- Yeni HTML/JS varsa: kullanıcı veya model kaynaklı metin `innerHTML`/`set:html` ile basılmaz (yalnız Astro'nun kendi render ettiği güvenilir içerik hariç).
- Yeni dış bağlantı: `rel="noopener noreferrer"`.
- Yeni yapılandırma: gizli değerler ortam dosyasından okunur, örnek dosyada boş bırakılır.

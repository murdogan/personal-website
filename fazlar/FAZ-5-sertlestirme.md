# Faz 5 — Sertleştirme ve Lansman

**Amaç:** Siteyi güvenlik, izleme, gizlilik ve kurtarma açısından canlıya hazır hâle getirmek ve `SITE_MODE=public` ile yayımlamak.
**Önkoşul:** Faz 4 çıkış kapısı.
**Referans:** V2 §14 (önbellek), §15 (güvenlik), §16 (gözlemlenebilirlik), §17.3 (CI kapıları), §18 (dağıtım ve kurtarma), D-01, D-09, D-10.

---

## İP-5.1 — CSP ve güvenlik başlıkları

**Amaç:** `script-src 'unsafe-inline'`'ı kaldırmak; satır içi betiklere sürüm başına hash ile izin vermek.

**Yapılacaklar:**

1. `web/scripts/csp-hashes.ts`: derlenmiş `dist/**/*.html` içindeki bütün satır içi `<script>` (src'siz) içeriklerinin SHA-256'sını (`'sha256-<base64>'`) hesaplar; tekil ve sıralı listeyi `out/csp/script-hashes.json`'a yazar. `just release` ve `just e2e` bunu çalıştırır.
2. `deployctl csp-snippet --hashes <file> [--frame-src <url>] [--prod]`: Caddy `header Content-Security-Policy "…"` satırını üretir:

   ```text
   default-src 'self'; script-src 'self' <hash'ler>; style-src 'self' 'unsafe-inline';
   img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-src <demo alanı veya 'none'>;
   frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'
   [--prod: ; upgrade-insecure-requests]
   ```

   `style-src 'unsafe-inline'` bilinçli olarak kalır (Astro'nun satır içi stilleri ve animasyonların stil öznitelikleri; risk düşük). Gerekçe `docs/adr/0011-csp.md`'ye yazılır.
3. CSP satırı `deploy/caddy/security-headers.caddy` dosyasından çıkarılır; artık sürüme özgüdür ve `active.caddy`'de üretilir. `deployctl package` hash'leri `release.json` → `cspScriptHashes`'e koyar; `deploy`, `active.caddy`'yi üretirken CSP satırını ekler. Yerel/CI Caddy yapılandırması aynı üreticiyle `out/csp/local.caddy` dosyasını üretip içe aktarır.
4. `web/tests/e2e/csp.spec.ts`: `securitypolicyviolation` olaylarını toplayan bir başlangıç betiği; ana sayfa (intro dahil), proje listesi/detay, arama (taklit API), asistan paneli (taklit SSE), sayfa geçişleri → **sıfır ihlal**.
5. Başlık denetimi: `headers.spec.ts`'e HSTS, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `COOP`, CSP'de `unsafe-inline`'ın `script-src`'de bulunmaması eklenir.
6. HSTS `preload` kararı ertelenir (geri dönüşü zor); ADR-0011'de not.

**Bitti sayılır:**

- [ ] CSP testinde ihlal yok; üretimde `curl -I` çıktısı raporda.

---

## İP-5.2 — İzleme, uyarılar ve log saklama

**Yapılacaklar:**

1. `deployctl monitor --once` (systemd timer ile 5 dk'da bir; `deploy/systemd/site-monitor.{service,timer}`):

   | Kontrol | Başarısızlık koşulu |
   | --- | --- |
   | Site | `https://<alan>/` 200 değil veya `release` meta değeri `active.json` ile farklı |
   | API | `/api/health/ready` 200 değil veya sürüm farklı |
   | Model | llama-server `/health` 10 dakikadan uzun süredir 200 değil |
   | Disk | `/srv` doluluk ≥ %80 |
   | Renk durumu | `active.color` çalışmıyor veya iki renk birden çalışıyor (dağıtım kilidi yokken) |

2. Uyarı: `/srv/site/config/monitor.toml` → yalnız `kind = "ntfy" | "generic-json"` ve tekrar aralığı (A-06). Webhook adresi gizli değerdir: `secrets.env` → `ALERT_WEBHOOK_URL`; `site-monitor.service` bunu `EnvironmentFile=/srv/site/config/secrets.env` ile alır (dosyayı systemd okur; servis kullanıcısının okuma izni gerekmez). Durum dosyası `/srv/site/monitor-state.json`: aynı sorun için en fazla 6 saatte bir tekrar; düzelince "düzeldi" bildirimi.
3. Harici uptime izleyici (kullanıcı kurar; ücretsiz bir servis): `/` ve `/api/health/ready`, ayrıca TLS sertifika süresi uyarısı. Runbook'a adımlar.
4. Log saklama ve gizlilik (İP-1.11'de kuruldu) doğrulanır: journald 14 gün; Caddy erişim logu günlük döner ve 7 gün tutulur (≈ en fazla 8 gün); erişim ve varsayılan logger'da sorgu dizesindeki `q`, `Referer` ve tam IP yok; site-api loglarında URI yok.
5. `just remote-status`: SSH ile `deployctl status` ve önemli metrikleri (aktif üretim, kuyruk reddi, model durumu) özetler.

**Testler:** `monitor` kontrolleri sahte sistemle (her koşul, tekrar bastırma, düzelme bildirimi).

**Bitti sayılır:**

- [ ] Test uyarısı kullanıcının kanalına ulaştı (llama-server 10 dk durdurularak veya eşik geçici düşürülerek).

---

## İP-5.3 — Gizlilik sayfası

**Yapılacaklar:**

1. `/gizlilik` sayfası; alt bilgide ve asistan panelinde bağlantı.
2. Taslak metin (kullanıcı onaylar; hukuki danışmanlık yerine geçmez):
   - Bu site çerez kullanmaz ve analitik/izleme aracı içermez.
   - IP adresiniz kötüye kullanımı önlemek için yalnız sunucu belleğinde, kısa süreli hız sınırlaması amacıyla kullanılır.
   - Web sunucusu erişim kayıtları, IP adresinin son kısmı maskelenmiş olarak en fazla 8 gün tutulur ve silinir. Bu kayıtlarda arama sorgularınız ve geldiğiniz sayfanın adresi bulunmaz.
   - Site asistanına yazdığınız sorular ve cevaplar sunucuda saklanmaz. Konuşma yalnız tarayıcınızın oturum belleğinde tutulur ve sekme kapanınca silinir.
   - Sorular, cevap üretmek için bu sunucuda çalışan yerel bir dil modeline gönderilir; üçüncü taraf bir yapay zekâ servisine gönderilmez.
   - Asistan cevapları site içeriğinden otomatik üretilir ve hata içerebilir.
   - İletişim: {e-posta}.
3. Metindeki her iddianın yapılandırmayla tutarlı olduğu bir kontrol listesiyle doğrulanır (çerez yok: tarayıcı depolamasında yalnız `sessionStorage` anahtarları; log saklama ayarları; asistanın dış servise gitmediği).

**Bitti sayılır:**

- [ ] Sayfa yayında; kontrol listesi raporda.

---

## İP-5.4 — Yük testleri

**Yapılacaklar:**

1. `site-eval load search --target http://127.0.0.1:<port> --rps 50 --duration 60s --spoof-xff`: sunucuda, etkin rengin doğrudan portuna (karşı uç 127.0.0.1 olduğu için `X-Forwarded-For` güvenilir; rastgele IP'lerle hız sınırına takılmadan ölçülür). Çıktı: p50/p95/p99, hata oranı.
2. `site-eval load assistant --target http://127.0.0.1:<port> --origin <origin> --spoof-xff --concurrency 20 --duration 120s`: sunucuda, etkin rengin doğrudan portuna; gerçek modelle kısa sorular. (Tek IP'den Caddy üzerinden yapılırsa ölçülen şey kabul semaforu değil 5/dk hız sınırı olur.) Doğrulananlar:
   - Kabul edilen eşzamanlı istek hiçbir anda `slots + queue_extra`'yı aşmıyor (metriklerden).
   - Reddedilenler 100 ms içinde 503 alıyor.
   - Test bitince `assistant_active_generations = 0` ve `/api/assistant/status` `available` (sızıntı yok).
3. Asistan yükü sürerken arama p95 < 100 ms (`CPUWeight` ayrımının etkisi).
4. Rapor `eval/reports/load-<tarih>.md`; sınır değerleri (V2 §10.6) gerekiyorsa ölçüme göre ayarlanır ve `docs/adr/0012-asistan-sinirlari.md`'ye yazılır.

**Bitti sayılır:**

- [ ] Rapor var; üç doğrulama da geçti.

---

## İP-5.5 — Runbook ve tatbikatlar

**Yapılacaklar:**

1. `deploy/RUNBOOK.md` tamamlanır: dağıtım, geri alma, model değiştirme, sunucuyu sıfırdan kurma, sertifika sorunu, takılmış kilit, disk dolması, llama-server bellek yetersizliği, uyarı gelince ilk bakılacaklar. Her bölüm kopyalanabilir komutlarla.
2. Tatbikat şablonu `docs/tatbikatlar/SABLON.md`: amaç, adımlar, başlangıç/bitiş zamanları, gözlemler, düzeltmeler.
3. Tatbikatlar (her biri `docs/tatbikatlar/<tarih>-<tür>.md`):
   - Geri alma (D-10).
   - Art arda iki push (D-11).
   - Model kapalı: site ve arama çalışıyor, asistan çevrim dışı, uyarı geliyor.
   - API çökmesi: etkin renk `kill -9` → systemd yeniden başlatıyor; sayfalar etkilenmiyor.
   - **Sıfırdan kurulum (D-01):** boş bir VM'de bootstrap → son sürümün dağıtımı → model kurulumu. Süre ölçülür; hedef RTO ≤ 2 saat. Eksik adımlar runbook'a eklenir.

**Bitti sayılır:**

- [ ] Beş tatbikat kayıtlı; sıfırdan kurulum süresi ölçüldü.

---

## İP-5.6 — Lansman

**Kontrol listesi:**

- [ ] Gerçek içerik tamam: profil, en az 3 proje, gerçek portre ve maske; hiçbir içerikte `placeholder: true` yok (`just check-public` geçiyor).
- [ ] `eval/search.toml` ve `eval/assistant.toml` gerçek içeriğe göre güncellendi; geri getirme eşiği ve üretim raporu kabul edildi.
- [ ] Bütün A-xx kararları kapandı (SORULAR.md).
- [ ] CSP ihlali yok; güvenlik başlıkları doğru; `cargo deny` ve `pnpm audit` temiz (istisnalar gerekçeli).
- [ ] Gizlilik sayfası onaylandı.
- [ ] İkinci uzak git kopyası (ayna) kuruldu (D-01).
- [ ] Uyarı kanalı ve harici uptime izleyici çalışıyor.
- [ ] GitHub değişkeni `SITE_MODE=public`; dağıtım yapıldı; `robots.txt` izin veriyor, sitemap erişilebilir, sayfalarda `noindex` yok.
- [ ] Search Console'a sitemap gönderildi (kullanıcı).
- [ ] Sosyal paylaşım önizlemeleri kontrol edildi (OG görselleri).
- [ ] Üretimde Lighthouse (mobil): LCP ≤ 2,5 sn, CLS ≤ 0,1, TBT ≤ 200 ms; erişilebilirlik ≥ 0,95.
- [ ] `PROJE_KAPSAMI.md` durum satırı güncellendi.

**Bitti sayılır:** Kontrol listesinin tamamı ✓.

---

## İP-5.7 — (İsteğe bağlı) Canlı demo altyapısı

İlk demo hazır olduğunda yapılır. Referans: V2 §13.

1. `demos.<alan-adı>` için ayrı Caddy site bloğu; statik demolar `/srv/demos/<slug>/` altından.
2. Demo başına ayrı, daha gevşek olabilecek CSP; ana sitenin `frame-src`'ine yalnız gömme gerekiyorsa demo alanı eklenir.
3. Sunucu gerektiren demo: ayrı systemd birimi, `MemoryMax`, `CPUQuota`, ayrı kullanıcı; ana API ve modelle kaynak paylaşmaz. Her biri için kısa ADR.
4. Demo dağıtımı ana sürüm hattından bağımsızdır (demonun kendi deposu/klasörü); ana site yalnız `demo.url` bağlantısını bilir.

---

## İP-5.8 — (İsteğe bağlı) Proje başına OG görseli

Derleme zamanında proje başlığı ve kapağından 1200×630 paylaşım görseli üretimi. Yeni bağımlılık gerektirir (ör. SVG → PNG dönüştürücü); AGENTS.md §9 gereği önce onay alınır.

---

## Faz 5 çıkış kapısı = Lansman

- [ ] İP-5.1 … İP-5.6 tamam.
- [ ] Lansmandan sonraki ilk hafta: günlük uyarı/metrik kontrolü; asistan için kullanıcı geri bildirimiyle gözlem (loglarda içerik olmadığı için yalnız metrikler ve kullanıcının kendi denemeleri).

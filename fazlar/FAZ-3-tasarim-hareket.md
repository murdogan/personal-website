# Faz 3 — Tasarım ve Hareket

**Amaç:** Dennis Snellenberg yönündeki görsel tasarımı, çok dilli intro'yu, sayfa geçişlerini ve portre arkasındaki etkileşimli akışı; performans, erişilebilirlik ve D-04 korunarak uygulamak.
**Önkoşul:** Faz 1 çıkış kapısı; İP-0.7 tercihleri (yoksa SORULAR.md varsayılanları).
**Referans:** [TASARIM.md](../TASARIM.md) (tamamı), taslaklar `referanslar/tasarim/03-…` ve `04-…`, V2 §12, §14.
**Kapsam dışı:** Asistan paneli (Faz 4; burada yalnız yer bırakılır).

Faz 2A–2B ile paralel yürüyebilir; **istisna:** İP-3.2 ve İP-3.5 arama adasını (`SearchBox`, `/ara` sonuçları) kullandığı için İP-2B.5'i bekler. 3.1, 3.3, 3.4, 3.6, 3.8 beklemeden yapılabilir. Ortak dosyalar (`BaseLayout.astro`, `SiteHeader.astro`, `tokens.css`) için birleştirme çakışmalarına dikkat edilir.

**Faz boyunca geçerli kural — yumuşak gezinme:** İP-3.7'den sonra sayfalar `ClientRouter` ile değişir ve paketlenmiş betikler yalnız bir kez çalışır. Bu fazdaki her sayfa betiği (menü düğmesi, hero girişi, önizleme değiştirici, akış başlatıcı) başlatma işini `document.addEventListener("astro:page-load", …)` içinde yapar ve `astro:before-swap`'ta kendini temizler. İP-3.7 öncesinde yazılan betikler de bu biçimde yazılır.

**Taslaklardan çıkarılan düzen (03 ve 04):**

- Üstte: solda isim (büyük harf, küçük boy), sağda `Projeler`, `Hakkımda`, `İletişim` ve hap biçimli arama alanı ("Sitede ara…").
- Hero (koyu, grafit): ortada portre; solda küçük büyük harfli etiket ("KİŞİSEL WEB SİTESİ") ve yuvarlak "aşağı" düğmesi; sağda ↘ oku ve 3 kısa satır ("Projelerim. Fikirlerim. Deneyimlerim."); en altta portrenin üstüne binen, ekran genişliğinde, ince ağırlıklı büyük isim ve sonunda uzun tire.
- Sağ kenarda yarısı ekrana yapışık "Bana sor" hapı (Faz 4).
- Açık bölüm (sıcak beyaz): küçük etiket ("SEÇİLİ ÇALIŞMALAR"), büyük başlık, solda tek önizleme görseli, sağda ince çizgilerle ayrılmış proje satırları (başlık, açıklama, "Canlı dene →", büyük ↗ oku).
- 04: portrenin arkasında sıcak beyaz ve sarı-yeşil yarı saydam akış çizgileri; bir kısmı omuzun önünden geçiyor; imlece yakın yerde çizgiler bükülüp toplanıyor.
- Taslaklardaki "ÖRNEK PORTRE" etiketi ve kişi yer tutucudur; üretime taşınmaz.

---

## İP-3.1 — Tasarım token'ları ve tipografi

**Yapılacaklar:**

1. `tokens.css`'i kesinleştir (A-08). Renkler İP-1.5 tablosundaki ölçülen değerlerdir; ek olarak:
   - `--hero-bg`: grafit radyal geçiş (merkez ≈ `#3a3b39`, kenar ≈ `#161817`; taslak 04'ten).
   - `--color-paper-50-a70`, `--color-paper-50-a40` gibi saydamlık türevleri (çizgiler, yer tutucu metin).
2. Tipografi (A-07, varsayılan Inter variable):

   | Token | Değer (başlangıç) | Kullanım |
   | --- | --- | --- |
   | `--font-sans` | Inter Variable, yedek: sistem sans | Hepsi |
   | `--text-display` | İsim için genişliğe göre hesaplanır (İP-3.3) | Büyük isim; ağırlık ≈ 300, harf aralığı −0.04em, satır yüksekliği 0.85, büyük harf |
   | `--text-h1` | `clamp(2.5rem, 4vw + 1rem, 5rem)`; ağırlık 400; −0.03em | Bölüm başlıkları ("Ürettiklerimi burada paylaşıyorum.") |
   | `--text-h2` | `clamp(1.75rem, 1.5vw + 1rem, 2.75rem)`; 400; −0.02em | Proje başlıkları |
   | `--text-lead` | `clamp(1.25rem, 1vw + 1rem, 1.75rem)`; 300 | Hero satırları |
   | `--text-body` | `clamp(1rem, 0.2vw + 0.95rem, 1.125rem)`; 400; 1.6 | Gövde |
   | `--text-eyebrow` | 0.75rem; 500; büyük harf; harf aralığı 0.22em | "KİŞİSEL WEB SİTESİ", "SEÇİLİ ÇALIŞMALAR" |
   | `--text-nav` | 0.875rem; 400 | Gezinme |

3. Yazı tipi yükleme: latin + latin-ext `woff2` önceden yüklenir; yedek yazı tipine `size-adjust`/`ascent-override` uygulanarak yazı tipi değişimindeki kayma azaltılır (değerler raporda).
4. Hareket token'ları: `--dur-fast: 150ms`, `--dur-base: 300ms`, `--dur-slow: 600ms`, `--dur-curtain: 700ms`, `--ease-out-expo`, `--ease-in-out-quart: cubic-bezier(0.76, 0, 0.24, 1)`.
5. Izgara ve boşluk: masaüstünde yatay kenar ≈ 40 px (`clamp(1rem, 3.3vw, 2.5rem)`), içerik en fazla 1440 px; bölüm dikey boşluğu `clamp(4rem, 10vw, 8rem)`.
6. Stil rehberi sayfası `/stil`: yalnız `SITE_MODE=prelaunch` derlemesinde üretilir (dinamik rota `getStaticPaths` ile). Renkler, yazı ölçeği, düğmeler, bağlantılar, etiketler, proje satırı ve uzun isimli hero örneği.

**Bitti sayılır:**

- [ ] `/stil` iki temada token'ları gösteriyor; `public` derlemesinde yok.
- [ ] Bütün metin/zemin çiftleri AA (axe); vurgu rengi açık zeminde metin olarak kullanılmıyor.
- [ ] Rapor, "i" içeren bir ismin büyük harfte "İ" göründüğünü (`lang="tr"`) gösteren ekran görüntüsünü içeriyor.

---

## İP-3.2 — Üst bölüm: gezinme ve arama alanı

**Yapılacaklar:**

1. Masaüstü (≥ 1024 px): yükseklik ≈ 80 px; solda isim (büyük harf, `--text-nav` biraz büyük); sağda gezinme (aralık 32 px) ve arama hapı (≈ 220 × 40 px, 1 px kenarlık `paper-50` %70, solda büyüteç, `--radius-pill`). Ana sayfada üst bölüm hero'nun üstüne biner (saydam zemin); diğer sayfalarda koyu bant.
2. A-12 varsayılanı (Dennis deseni): üst bölüm sabit değildir. Sayfa hero yüksekliği kadar kaydırılınca sağ üstte 56 px'lik yuvarlak düğme belirir (ölçeklenerek; azaltılmış harekette anında). Düğme bir `<dialog>` açar: gezinme bağlantıları + arama alanı (açılınca arama alanına odak). `Esc` kapatır, odak düğmeye döner.
3. Mobil (< 768 px): solda isim, sağda her zaman görünen yuvarlak menü düğmesi; diyalog tam ekran; arama alanı en üstte ve belirgin.
4. `SearchBox` (İP-2B.5) iki yerde kullanılır (üst bölüm ve diyalog); `idPrefix` prop'u ile kimlik çakışması önlenir.
5. `/` kısayolu, görünür bir arama alanı yoksa diyaloğu açıp aramaya odaklanır.

**Testler (Playwright):** masaüstü ve mobilde arama alanına ulaşım; diyalog klavye akışı (odak tuzağı, `Esc`, odağın dönmesi); `/` kısayolu; axe.

**Bitti sayılır:**

- [ ] Taslak 03 ile yan yana karşılaştırma ekran görüntüsü raporda.
- [ ] Arama, her ekran genişliğinde en fazla bir dokunuşla erişilebilir.

---

## İP-3.3 — Ana sayfa hero kompozisyonu

**Profil şeması güncellemesi** (İP-1.3 şemasına ek): `portrait.src` artık **arka planı ayrılmış (alfa kanallı)** portredir; isteğe bağlı `portrait.mask` (ön akış maskesi, İP-3.9). Yer tutucu siluet ve yer tutucu maske `make-placeholders.ts` ile üretilir.

**Düzen (masaüstü):**

- Bölüm `data-theme="dark"`, zemin `--hero-bg`, `min-height: 100svh` (en az 640 px).
- Katmanlar (arkadan öne, V2 §12.4): `[data-flow-layer="back"]` (boş, İP-3.9 doldurur) → portre → `[data-flow-layer="front"]` → isim. Boş katmanlar şimdiden yerinde, boyutları portre kutusuna bağlı.
- Portre: yatay ortada, alta yaslı, hero yüksekliğinin ≈ %90'ı; `<picture>` (AVIF, WebP), genişlikler 480/720/960/1280/1600, `sizes` uygun; `fetchpriority="high"`, `loading="eager"`. **LCP öğesi budur.**
- Sol blok (≈ %40 yükseklikte): `siteLabel` (eyebrow) ve 60 px yuvarlak "aşağı" bağlantısı (`#calismalar`, erişilebilir ad "Çalışmalara in").
- Sağ blok (≈ %40 yükseklikte): ↘ simgesi (dekoratif) ve `heroLines` (alt alta, `--text-lead`).
- İsim: `<h1>`, en altta, tam genişlik, portrenin üstünde; sonunda dekoratif uzun tire (`aria-hidden`). Yazı boyu isim uzunluğuna göre: derleme zamanında `--name-chars` (karakter sayısı + tire) CSS değişkenine yazılır; `font-size: min(calc((100vw - 2 * var(--gutter)) / (var(--name-chars) * var(--glyph-ratio))), 22rem)`; `--glyph-ratio` başlangıçta 0.62, görsel olarak ayarlanır.
- Mobil (< 768 px, öneri; Faz 3 incelemesinde onaylanır): portre ≈ 60svh; isim portrenin altına biner ama daha küçük; `siteLabel` ve `heroLines` hero'nun altında; yuvarlak düğme gizli.
- Giriş hareketi (intro yoksa da): isim satırı 40 px aşağıdan yukarı kayarak ve belirerek gelir (600 ms, `--ease-out-expo`); azaltılmış harekette yok. JS yoksa isim doğrudan görünür.

**Testler:** Playwright 320, 375, 768, 1024, 1440, 1920 genişliklerinde yatay taşma yok (`scrollWidth <= innerWidth`); `/stil` sayfasındaki 24 karakterlik isim örneğinde de taşma yok; LCP öğesinin portre olduğu (`PerformanceObserver` ile) doğrulanıyor.

**Bitti sayılır:**

- [ ] Taslak 03 ile yan yana ekran görüntüsü raporda.
- [ ] LCP ≤ 2,5 sn (Lighthouse mobil), CLS ≤ 0,1.

---

## İP-3.4 — Proje satırları ve proje listesi

**Taslak yorumu (onaya tabi):** 03/04'te görsel yalnız bir kez, solda görünüyor. Bu, Dennis'teki gibi **tek önizleme alanı** olarak uygulanır: masaüstünde solda yapışkan (sticky) bir önizleme, üzerine gelinen veya klavyeyle odaklanan satırın kapağını gösterir. İncelemede beğenilmezse her satır kendi görseliyle gösterilir (ikinci düzen de bileşende seçenek olarak bulunur).

**Yapılacaklar:**

1. Bölüm `data-theme="light"`, `id="calismalar"`: `projectsEyebrow` + `projectsHeading` (`--text-h1`).
2. Masaüstü ızgarası: `[önizleme ≈ %32] [satırlar]`, aralık ≈ 40 px. Önizleme 6:5 oranlı, `object-fit: cover`.
3. Satır: başlık (`h3`, `--text-h2`, detaya bağlantı), özet (`stone-600`), varsa "Canlı dene →" (altı çizili), sağda büyük ↗ oku (detaya bağlantı; erişilebilir ad "{başlık} projesini incele"). Satırlar arasında yalnız satır sütununu kaplayan ince çizgi (`paper-200`); listenin altında tam genişlik çizgi.
4. Önizleme değişimi: küçük vanilla betik (React değil); `pointerenter` ve `focusin` ile; 300 ms çapraz geçiş; azaltılmış harekette anında. JS yoksa ilk projenin kapağı görünür.
5. Üzerine gelince: ok 4 px sağ-yukarı kayar, başlığın altı çizgisi büyür; azaltılmış harekette yok.
6. Mobil: önizleme alanı yok; her satır kendi kapağını üstte gösterir.
7. Öne çıkan projelerden fazlası varsa listenin altında "Tüm projeler →" (`/projeler`).
8. `/projeler` sayfası: aynı satır bileşeni, bütün projeler; üstte etiket filtre bağlantıları (etkin etiket `aria-current`).

**Bitti sayılır:**

- [ ] Taslakla yan yana ekran görüntüsü; klavyeyle gezinirken önizleme değişiyor.
- [ ] Görseller CLS üretmiyor.

---

## İP-3.5 — Proje detayı, Hakkımda, alt bilgi ve 404

**Yapılacaklar:**

1. **Proje detayı:** koyu üst bant içinde başlık (`--text-h1`) ve meta satırı (rol, yıl, durum, etiketler; eyebrow biçiminde); tam genişlik kapak; gövde sütunu ≤ 70ch; Markdown öğeleri için tipografi (başlıklar, listeler, kod, alıntı, tablo, `Figure`); "Kaynak kod" ve "Canlı dene" hap düğmeleri; altta önceki/sonraki proje (büyük satırlar).
2. **Hakkımda:** başlık; gövde metni; yetenekler etiket listesi; CV düğmesi; iletişim bağlantıları.
3. **Alt bilgi** (taslak 01/02'den): koyu; büyük "İletişime geç." başlığı ve sarı-yeşil ↗; sağda altı çizili E-posta, GitHub, LinkedIn; en altta ince çizgi, isim ve yıl. `id="iletisim"` korunur.
4. **404:** koyu; büyük "404"; kısa mesaj; arama alanı; "Projelere dön" bağlantısı.
5. `/ara` sayfası ve arama sonuçları (İP-2B.5) İP-1.5'teki geçici stilden bu dile taşınır.

**Bitti sayılır:**

- [ ] Bütün sayfalar aynı görsel dili kullanıyor; axe temiz; ekran görüntüleri raporda.

---

## İP-3.6 — Çok dilli intro ve emniyet

**Referans:** V2 §12.3 (emniyet kuralları bağlayıcıdır), TASARIM §5, A-10.

**Yapılacaklar:**

1. `web/src/config/intro.ts`: selamlar ve diller (A-10 varsayılanı): `[{ text: "Merhaba", lang: "tr" }, { text: "Hello", lang: "en" }, { text: "Bonjour", lang: "fr" }, { text: "Hola", lang: "es" }, { text: "Ciao", lang: "it" }, { text: "Hallo", lang: "de" }]`; ilk selam ≈ 400 ms, diğerleri ≈ 200 ms; selamlar toplam ≤ 1,8 sn, örtünün kalkışıyla birlikte ≤ 2,5 sn (CSS bekçisinin 3 sn'sinden kısa).
2. `web/src/islands/intro/IntroOverlay.astro`: varsayılan gizli; `aria-hidden="true"`; koyu tam ekran; ortada küçük bir vurgu noktası ve selam metni (`--text-h1`); her selam kendi `lang` özniteliğiyle.
3. `<head>` içinde satır içi betik (`is:inline`, olabildiğince küçük; Vite'ın hash'li modüllerini içe aktaramaz, bu yüzden yalnız emniyet ve atlamayı yapar):
   - `sessionStorage` erişimi `try/catch` içinde; erişilemezse intro gösterilmez.
   - Oturumda ilk ziyaretse: önce JS yedek zamanlayıcısı (3,5 sn) kurulur, sonra `<html class="intro">` eklenir, bayrak yazılır.
   - Atlama işleyicileri (`pointerdown`, `keydown`) **yalnız `html.intro` varken** çalışır: sınıfı kaldırır, odağı `#main`'e taşır ve kendilerini hemen kaldırır. Intro bittikten sonra hiçbir tuş veya tıklama odağı değiştirmez.
4. Paketlenmiş bir `<script>` (normal Astro betiği) yalnız `html.intro` varsa animasyon modülünü `import()` ile yükler; `import()` reddedilirse veya modül hata verirse sınıfı hemen kaldırır.
5. CSS bekçisi (V2 §12.3 taslağı): `html.intro .intro-overlay { display: grid; animation: intro-guard 0s linear 3s forwards; }` ve `@keyframes intro-guard { to { visibility: hidden; } }`.
6. Animasyon modülü (`web/src/islands/intro/intro.ts`, GSAP): selamları sırayla değiştirir; sonunda örtü yukarı kayar (`translateY(-100%)`, 700 ms, `--ease-in-out-quart`), sınıfı kaldırır, hero giriş hareketini tetikler. `prefers-reduced-motion`: selam geçişleri yerine tek kısa solma (≤ 300 ms).
7. İçerik ve portre örtünün altında normal yüklenir; intro hiçbir kaynağı bekletmez.

**Testler (Playwright):**

- Intro bittikten sonra `Tab` ile gezinirken odak `#main`'e sıçramıyor.

- İlk ziyarette örtü görünür ve ≤ 2,8 sn içinde kalkar; ikinci ziyarette görünmez.
- Animasyon modülü isteği engellenince (`page.route(… abort)`) örtü ≤ 3,2 sn içinde kalkar ve sayfa tıklanabilir.
- Herhangi bir tuş örtüyü hemen kaldırır, odak `#main`'de.
- JS kapalıyken örtü hiç görünmez.
- `reducedMotion: "reduce"` ile örtü ≤ 500 ms içinde kalkar.
- `sessionStorage` engellendiğinde (betikle `throw` taklidi) intro gösterilmez.

**Bitti sayılır:**

- [ ] Bütün testler geçiyor; LCP intro yüzünden kötüleşmiyor (Lighthouse karşılaştırması raporda).

---

## İP-3.7 — Sayfa geçişleri

**Referans:** V2 §12.2, §7.1.

**Yapılacaklar:**

1. `BaseLayout`'a `<ClientRouter />`.
2. Perde geçişi: koyu bir perde öğesi; gezinme başlarken alttan yukarı örter (≈ 500 ms), yeni sayfa takılınca yukarı doğru çekilir. Astro yaşam döngüsü olaylarıyla (`astro:before-preparation` içinde yükleyiciyi animasyonu bekleyecek biçimde sarmak, `astro:after-swap` / `astro:page-load` ile açmak). Bağlantıda `data-transition-label` varsa perdede kısa etiket gösterilir.
3. Sürüm farkı: `astro:before-swap`'ta yeni belgenin `<meta name="release">` değeri mevcut sayfadan farklıysa takas, tam sayfa yüklemesine çevrilir (ör. `event.swap` yerine `location.assign(hedef)`). Kullanılan Astro sürümünün olay API'sine göre doğrulanır.
4. Başlatma ve temizlik: sayfa betikleri `astro:page-load`'da başlar (faz girişindeki kural); GSAP animasyonları bir `gsap.context()` içinde ve `astro:before-swap`'ta `revert()`; olay dinleyicileri `AbortController` ile kaldırılır.
5. Parça yükleme hatası (V2 §7.1): `window`'da `vite:preloadError` olayı ve dinamik `import()` reddi yakalanır; sayfa **bir kez** tam yeniden yüklenir (`sessionStorage` bayrağıyla döngü engellenir; ikinci hatada yalnız konsola yazılır ve ada sessizce devre dışı kalır).
6. Erişilebilirlik: takas sonrası odak `<main>`'e; Astro'nun rota duyurucusu açık kalır.
7. Azaltılmış hareket: perde yok; en fazla 150 ms opaklık.
8. Üst bölüm kalıcı yapılmaz (etkin bağlantı değişir). Asistan adası Faz 4'te `transition:persist` alır.

**Testler (Playwright):** sayfalar arası gezinme ve geri/ileri; içerik doğru güncelleniyor; odak `main`'de; konsol hatası yok; **gezinmeden sonra** menü düğmesi, proje önizleme değiştirici ve hero girişi çalışıyor; azaltılmış harekette perde yok; sürüm farkı taklidinde (bir sonraki sayfanın HTML'i `page.route` ile meta değeri değiştirilerek) tam sayfa yüklemesi oluyor (pencereye konan işaret kayboluyor); bir ada parçasının isteği 404'e çevrilince sayfa bir kez yeniden yükleniyor ve döngüye girmiyor.

**Bitti sayılır:**

- [ ] Testler geçiyor; geçişler 60 fps'e yakın (DevTools performans kaydı raporda özetlenir).

---

## İP-3.8 — Hero akışı prototipleri ve ölçüm

**Amaç:** A-03 kararını ölçümle vermek. Referans: TASARIM §6, taslak 04, V2 §12.4.

**Yapılacaklar:**

1. `web/src/islands/hero/flow/`:
   - `types.ts`: `FlowRenderer { mount(back, front, opts); setPointer(x, y, active); setQuality(tier); pause(); resume(); destroy(); }`.
   - `model.ts`: ortak akış modeli — 12–24 şerit; her şerit zamanla kayan sinüs/gürültü toplamı bir eğri; imleç yakınında eğrilerin imlece doğru bükülüp toplanması (taslak 04'teki "sıkışma"); şeritlerin ≈ %25'i ön katmana atanır. Renkler `paper-50` ve `accent-300` (yarı saydam, toplamalı karışım).
   - `pointer.ts`: imleç düz bir nesneye yazılır; yumuşatma (gecikmeli takip) döngüde; fare durunca etki sönümlenir.
   - `perf.ts`: kare süresi örnekleyici (p50/p95, düşen kare), kalite kademesi yöneticisi.
   - `webgl/` (Prototip A): WebGL2; şeritler vertex shader'da üretilen üçgen şeritleri; iki tuval aynı modeli farklı alt kümelerle çizer.
   - `canvas2d/` (Prototip B): aynı model CPU'da çoklu çizgi olarak hesaplanır, `lighter` karışımıyla çizilir.
2. Seçim: `?flow=a`, `?flow=b`, `?flow=off`. `?debug=perf` ekranda fps, p50/p95 kare süresi, kademe, DPR ve çiziciyi gösterir; 10 sn sonra konsola JSON özet yazar.
3. Ölçüm protokolü (`docs/adr/0009-hero-akis-teknigi.md` içinde):
   - Cihazlar: masaüstü (tümleşik GPU'lu dizüstü) ve orta seviye bir Android telefon (Chrome uzaktan hata ayıklama).
   - Senaryolar: 10 sn boşta, 10 sn imleç hareketi, kaydırma sırasında, arama alanına yazarken.
   - Ölçütler: p95 kare süresi; ≥ 50 ms uzun görevler; yazma sırasında girdi gecikmesi; DevTools CPU kullanımı.
   - Kabul: masaüstünde en yüksek kademede p95 ≤ 16,7 ms; mobilde en az bir kademede p95 ≤ 33 ms; çizicinin neden olduğu uzun görev yok; yazma gecikmesi akış kapalı duruma göre ≤ 20 ms fazla.
4. ADR-09: ölçüm tablosu, ekran görüntüleri, öneri. Karar kullanıcıdadır (A-03).

**Bitti sayılır:**

- [ ] İki prototip çalışıyor; ADR-09 ölçümlerle dolduruldu; kullanıcıya karar sunuldu.

---

## İP-3.9 — Hero akışı üretim sürümü

**Önkoşul:** A-03 kararı (teknik seçildi). Gerçek portre ve maske yoksa yer tutucularla tamamlanır; gerçekleri gelince yalnız varlıklar değişir.

**Yapılacaklar (V2 §12.4 kuralları bağlayıcı):**

1. Seçilen çizici kalır; diğeri silinir (ADR-09'a not).
2. Yükleme: modül `load` olayından ve `requestIdleCallback`'ten (en geç 2 sn) sonra, yalnız şu koşullarda dinamik olarak yüklenir: azaltılmış hareket kapalı, `navigator.connection.saveData` değil, `?flow=off` değil, hero görünür.
3. Ön katman maskesi: `mask-image` portre kutusuyla aynı en-boy oranında, `mask-size: 100% 100%`; maske görseli Astro ile optimize edilir.
4. Kalite kademeleri: 3 (tam), 2 (şerit ve DPR × 0,75), 1 (en az şerit, DPR × 0,5), 0 (statik). İlk 60 karede p95 ölçülür; bütçe aşılırsa bir kademe düşülür; kademe 0'da tuvaller kaldırılır.
5. Duraklatma: hero görünmezken (IntersectionObserver) ve sekme gizliyken (`visibilitychange`). Dokunmatik cihazda (`(hover: none)`) imleç yerine yavaş otomatik yol.
6. `webglcontextlost` → çizici yok edilir, statik portre kalır.
7. Akış hiçbir zaman portre yüzünü, gözleri ve ismi kapatmaz (ön katman maskesi yalnız omuz/gövde bölgesinde).
8. Sayfa geçişinde (`astro:before-swap`) `destroy()`.

**Testler (Playwright):** masaüstünde tuvaller oluşuyor; azaltılmış harekette ve `?flow=off`'ta oluşmuyor; WebGL devre dışıyken (çizici A seçildiyse) statik portreye düşülüyor; LCP öğesi hâlâ portre; sayfa geçişinden sonra animasyon döngüsü sayısı artmıyor (sızıntı yok).

**Bitti sayılır:**

- [ ] Testler geçiyor; Lighthouse bütçeleri tutuyor; gerçek cihazda ölçüm raporda.

---

## İP-3.10 — Mobil, erişilebilirlik ve performans

**Yapılacaklar:**

1. Duyarlı kontrol: 320, 375, 414, 768, 1024, 1280, 1440, 1920 px. Dokunma hedefleri ≥ 44 px. `forced-colors` modunda kenarlıklar ve odak görünür.
2. Klavye ile bütün akışlar; ekran okuyucuyla ana sayfa ve proje detayı (Narrator/NVDA; rapor gözlemleri içerir).
3. JS bütçesi, HTML ayrıştırarak değil **ağ ölçümüyle** denetlenir (Astro adaları `astro-island` öznitelikleri ve dinamik `import()` ile yüklendiği için HTML'deki betik etiketleri eksik sayar): `web/tests/e2e/js-budget.spec.ts` Caddy üzerinden her ana sayfayı açar, aktarılan bütün JS yanıtlarının (sıkıştırılmış) boyutlarını toplar:
   - `load` olayına kadar (ilk yük; intro ve hero akışı modülleri bu aşamada yüklenmez): ≤ 100 KB.
   - Ağ boşalana kadar (dinamikler dahil): ≤ 180 KB.
4. Lighthouse `resource-summary:script:size` eşiği bu fazda bilinçli olarak 184320 bayta (180 KB) güncellenir, çünkü Lighthouse ertelenmiş hero modülünü de ölçer; ilk yük sınırı (100 KB) yukarıdaki testte korunur. Diğer Lighthouse eşikleri (İP-1.7) aynen kalır.
5. `just screenshots`: ana sayfa, proje listesi, proje detayı, hakkımda, 404 için 375 / 768 / 1440 px ekran görüntüleri (`out/screenshots/`); tasarım incelemesinde kullanılır.

**Bitti sayılır:**

- [ ] Bütçeler ve axe CI'da geçiyor; ekran görüntüleri üretildi.

---

## Faz 3 çıkış kapısı

- [ ] Kullanıcı tasarım incelemesi: `just screenshots` çıktıları ve canlı site taslak 03/04 ile karşılaştırıldı; onaylanan/değişecek noktalar SORULAR.md'ye yazıldı.
- [ ] A-03, A-07, A-08, A-10, A-12 kapandı.
- [ ] D-04: JS kapalıyken, WebGL yokken ve intro modülü yüklenemezken site kullanılabilir (testlerle).
- [ ] Lighthouse: LCP ≤ 2,5 sn, CLS ≤ 0,1, TBT ≤ 200 ms (mobil); erişilebilirlik ≥ 0,95.
- [ ] Gerçek telefonda elle kontrol yapıldı.

**Faz 4'e devir:** Asistan için sağ kenarda "Bana sor" hapının yeri ve z-index katmanı tanımlı (`--z-assistant`); `transition:persist` kullanılmaya hazır.

# Faz 0 — Hazırlık

**Yürüten:** Kullanıcı. Bu fazda kod yazılmaz.
**Amaç:** Codex'in Faz 1'e başlayabilmesi için kararları, araçları, depoyu ve girdileri hazırlamak.
**Çıktı:** Kurulu geliştirme makinesi, boş bir GitHub deposu, doldurulmuş sunucu bilgisi ve içerik hazırlığının başlaması.

Faz 1'e başlamak için **0.1, 0.2 ve 0.3 zorunludur.** Diğerleri, tabloda belirtilen İP'lere kadar tamamlanabilir.

---

## İP-0.1 — Temel kararların onayı

[TEKNIK_MIMARI_V2.md](../TEKNIK_MIMARI_V2.md) §2.2'deki kararlar bu fazlarla uygulanacak. Özellikle:

- [ ] ADR-01: İçerik git'te Markdown/MDX; yönetim paneli yok.
- [ ] ADR-02: v1'de veritabanı yok.
- [ ] ADR-03: Astro statik çıktı + React adaları (Next.js yerine).
- [ ] ADR-05: Model olarak llama.cpp `llama-server`.
- [ ] Kod yazımı AI ajanıyla, İP birimleriyle; push ve dağıtım onayı kullanıcıda.

İtiraz varsa Faz 1'den önce V2 güncellenir; fazlar buna göre yeniden yazılır.

---

## İP-0.2 — Geliştirme makinesi kurulumu (Windows)

| Araç | Not | Doğrulama |
| --- | --- | --- |
| Git | Satır sonu: depo `.gitattributes` ile LF kullanır | `git --version` |
| Visual Studio Build Tools | "Desktop development with C++" iş yükü (Rust MSVC bağlayıcısı için) | — |
| rustup | MSVC toolchain; sürümü depo sabitler | `rustup --version` |
| Node.js LTS | Güncel LTS | `node --version` |
| pnpm | `corepack enable` ile; sürümü depo sabitler | `pnpm --version` |
| just | `winget install Casey.Just` veya `cargo install just` | `just --version` |
| Caddy | Resmî sürüm sayfasından Windows binary'si, PATH'e eklenir (yerel e2e testleri için) | `caddy version` |
| cargo-deny | `cargo install cargo-deny --locked` | `cargo deny --version` |
| VS Code eklentileri | rust-analyzer, Astro, ESLint, Prettier, Even Better TOML | — |
| Codex | Kendi kurulum belgesine göre; depo kökünde çalıştırılır | — |

Playwright tarayıcıları İP-1.7'de `pnpm exec playwright install` ile kurulur. Git ile gelen Git Bash, betiklerin `bash -n` sözdizimi denetimi için yeterlidir; shellcheck yalnız CI'da çalışır.

Windows notu: Windows PowerShell 5.1'de `curl` bir takma addır (Invoke-WebRequest); belgelerdeki örneklerde `curl.exe` kullanılır. `VAR=x komut` ve `a && b` biçimleri çalışmaz; ortam değişkeni gereken her şey `just` recipe'leri üzerinden çalıştırılır.

---

## İP-0.3 — Git deposu ve GitHub

1. Bu klasörde (`PW`) `git init`, varsayılan dal `main`.
2. GitHub'da **özel** bir depo oluştur ve uzak olarak ekle.
3. Mevcut belgeleri ve `referanslar/` klasörünü ilk commit olarak ekle.
4. Dal koruması (`main`): doğrudan push serbest kalabilir (tek geliştirici), ama İP-1.12'den sonra "CI geçmeden birleştirme yok" kuralı açılır.
5. İkinci bir uzak kopya (yedek) Faz 5'te eklenir (İP-5.6).

Not: GitHub dışında bir sağlayıcı seçilirse A-04'e yazılır; CI/CD İP'leri ona göre uyarlanır.

---

## İP-0.4 — Sunucu bilgileri (A-01)

Aşağıdakileri [SORULAR.md](SORULAR.md) A-01 satırına yaz:

```text
İşletim sistemi ve sürümü:
CPU modeli, çekirdek/iş parçacığı, mimari (x86_64 / arm64):
RAM:
GPU ve VRAM (yoksa "yok"):
Disk tipi ve boş alan:
Public IPv4/IPv6, ağ hızı:
Sunucuda çalışan başka servisler / açık portlar:
Root/sudo erişimi var mı:
Sunucu fiziksel mi, VPS mi (sağlayıcı):
```

Bu bilgi model kısa listesini (Faz 4), derleme hedefini (x86_64/arm64) ve kurulum betiklerini belirler.

Sunucu hazır değilse Faz 1–3'ün kod İP'leri ilerler; yalnız İP-1.13 ve sunucu adımları bekler.

---

## İP-0.5 — Alan adı ve DNS (A-04)

- [ ] Alan adı alındı.
- [ ] `A` (ve varsa `AAAA`) kaydı sunucuya yönlendirildi.
- [ ] İleride canlı demolar için `demos.<alan-adı>` alt alanı ayrılacak (şimdi kayıt gerekmez).
- [ ] CDN kullanılmayacak (varsayılan). Kullanılacaksa A-04'e yaz.

---

## İP-0.6 — İçerik ve portre hazırlığı

Lansmana kadar gerekir; Faz 1 yer tutucu içerikle başlar. Hazırlandıkça `content/` altına eklenir (Faz 1 sonrası şablonlar hazır olur).

### Profil (`content/profile.md`)

- Ad soyad, kısa unvan (headline), 1–3 cümlelik kısa tanıtım (≤ 300 karakter).
- Hero satırları (tasarımdaki "Projelerim. Fikirlerim. Deneyimlerim." gibi; 2–4 kısa satır).
- Yetenekler listesi.
- E-posta, GitHub, LinkedIn, CV (PDF) bağlantıları.
- Hakkımda metni (Markdown).

### Projeler (`content/projects/<slug>/index.mdx`)

Her proje için:

- Başlık, tek cümlelik özet (≤ 160 karakter), rolün/katkın, yıl, durum.
- Teknoloji etiketleri (`content/tags.yaml`'daki etiketlerden).
- Kapak görseli (yatay; en az 1200 px, önerilen 1600 px genişlik) ve alt metni.
- Gövde metni; önerilen başlıklar: **Problem**, **Çözüm**, **Katkım**, **Teknik notlar**, **Sonuç**.
- Kaynak kod bağlantısı (varsa), canlı demo bağlantısı (varsa).

Asistanın iyi cevap vermesi için gövde metninde gerçek bilgiler açıkça yazılmalı: hangi problemi çözdün, hangi teknolojiyi neden seçtin, senin payın neydi.

### Portre

- Yüksek çözünürlüklü (en az 2400 px yükseklik), sade arka planlı, iyi ışıklı fotoğraf.
- Arka planı ayrılmış (şeffaf PNG) sürümü: İP-3.3'te gerekir. Bir dekupaj aracıyla hazırlanabilir; kenar kalitesi elle kontrol edilmeli.
- Öne geçen akış maskesi (omuz/gövde bölgesi, siyah-beyaz PNG): İP-3.9'da gerekir; Codex yer tutucu maske üretir, gerçek portreye göre elle ayarlanır.
- AI ile üretilmiş örnek portre üretimde **kullanılmaz**.

---

## İP-0.7 — Tasarım tercihleri (A-07…A-12)

Faz 3 başlamadan önce [SORULAR.md](SORULAR.md)'deki A-07 (yazı tipi), A-08 (renkler), A-10 (intro), A-11 (sohbet paneli), A-12 (üst gezinme) satırlarına tercihini yaz. Yazmazsan tablodaki varsayılanlarla ilerlenir ve Faz 3 incelemesinde değiştirilebilir.

---

## Faz 0 çıkış kapısı

- [ ] İP-0.1 onaylandı.
- [ ] İP-0.2: tablodaki bütün doğrulama komutları çalışıyor.
- [ ] İP-0.3: depo GitHub'da, ilk commit gönderildi.
- [ ] A-01 ve A-04 için en azından tahmini bilgi yazıldı (kesin değilse "tahmini" notuyla).

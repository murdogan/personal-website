# Açık Kararlar ve Sorular

Bu dosya iki şeyi tutar: (1) kullanıcının vermesi gereken kararlar, (2) ajanın çalışırken sorduğu sorular. Kapanan bir kararın cevabı ve tarihi yazılır; ilgili İP o cevaba göre uygulanır.

Durum değerleri: `Açık`, `Varsayılanla ilerleniyor`, `Kapandı`.

## 1. Kullanıcı kararları

| No | Karar | Gerektiği yer | Varsayılan (karar verilmezse) | Durum | Cevap / tarih |
| --- | --- | --- | --- | --- | --- |
| A-01 | Sunucu: OS, CPU (çekirdek, mimari x86_64/arm64), RAM, GPU/VRAM, disk, ağ; sunucuda başka servis var mı | İP-1.11, İP-1.13, Faz 4 | Debian/Ubuntu LTS, x86_64, GPU yok | Açık | |
| A-02 | Model seçimi (kısa listeden, ölçümle) | İP-4.7 sonu | — (ölçüm zorunlu) | Açık | |
| A-03 | Hero akış tekniği (prototip ölçümüyle); gerçek portre ve maske | İP-3.8, İP-3.9 | Ölçümde kazanan teknik; yer tutucu siluet | Açık | |
| A-04 | Alan adı, DNS sağlayıcısı, CDN kullanımı; depo ve CI sağlayıcısı | İP-1.12, İP-1.13 | GitHub + GitHub Actions; CDN yok | Açık | |
| A-05 | Asistan gecikme hedefleri (ilk token, toplam) | İP-4.7 | İlk token ≤ 8 sn, toplam ≤ 45 sn (ölçümle güncellenir) | Açık | |
| A-06 | Uyarı kanalı (ör. ntfy, Telegram botu, e-posta) | İP-5.2 | Webhook URL'si; kanal kullanıcı seçer | Açık | |
| A-07 | Yazı tipi | İP-3.1 | Inter (variable, OFL, Türkçe karakterli). Lisanslı alternatif: PP Neue Montreal (Dennis'e en yakın) | Açık | |
| A-08 | Renk token'larının onayı (taslaklardan ölçülen değerler, İP-3.1) | İP-3.1 | Ölçülen değerler | Açık | |
| A-09 | Asistanın anlatım kişisi | İP-4.3 | Üçüncü şahıs ("Ad Soyad şu projede…"); asistan kişinin ağzından konuşmaz | Açık | |
| A-10 | Intro: diller, sıra, süre | İP-3.6 | Merhaba → Hello → Bonjour → Hola → Ciao → Hallo; selamlar ≤ 1,8 sn, örtünün kalkışıyla ≤ 2,5 sn | Açık | |
| A-11 | Sohbet panelinin ilk açılış durumu ve mobil konumu | İP-4.6 | Masaüstü ve mobilde kapalı başlar; mobilde alttan açılan panel | Açık | |
| A-12 | Üst gezinmenin kaydırmada davranışı | İP-3.2 | Üst bölüm sabit değil; hero geçildikten sonra sağ üstte yuvarlak menü/arama düğmesi belirir (Dennis deseni) | Açık | |
| A-13 | Lansman zamanı (`SITE_MODE=public` geçişi) | İP-5.6 | — | Açık | |
| A-14 | Proje durum değerleri ve demo açılma biçimi | İP-1.3, İP-1.4 | Durum: `tamamlandi`, `suruyor`, `arsiv`; "Canlı dene" yeni sekmede açılır | Açık | |

## 2. Ajan soruları

Ajan, çalışırken ürün/tasarım/güvenlik kararı gerektiren bir belirsizlikle karşılaşırsa buraya ekler ve kullanıcıya sorar.

Biçim:

```text
### S-<sıra> (İP-x.y) Kısa başlık
Bağlam: ...
Seçenekler: (a) ... (b) ...
Önerim: ...
Durum: Açık
Cevap:
```

(Henüz soru yok.)

# Kişisel Web Sitesi — Proje Kapsamı

Son güncelleme: 26 Ekim 2026

Durum: İçerik ve tasarım üzerinde çalışılıyor. Uygulama kodu henüz oluşturulmadı. Bu belge konuşmada belirlenen kapsamı ve açık kararları kaydeder; bir teknik uygulama planı değildir.

Teknik mimari, önerilen teknoloji temeli ve uygulama fazları ayrı [TEKNIK_MIMARI.md](TEKNIK_MIMARI.md) dosyasındadır. Görsel yön [TASARIM.md](TASARIM.md) dosyasında tutulur.

## 1. Projenin amacı

Kendimi tanıtabildiğim, yaptığım projeleri sergileyebildiğim ve zamanla farklı teknolojilerle geliştirdiğim özellikleri ekleyebildiğim bir kişisel web sitesi oluşturmak.

Ziyaretçi sitede kim olduğumu öğrenebilmeli, çalışmalarımı inceleyebilmeli, uygun projelerin canlı demolarını deneyebilmeli ve site içeriğine arama ya da yapay zekâ asistanı üzerinden ulaşabilmeli.

## 2. Planlanan içerik

| Bölüm | İçerik |
| --- | --- |
| Ana sayfa | Kısa kişisel tanıtım, öne çıkan projeler ve temel bölümlere erişim. |
| Hakkımda | Deneyim, yetenekler, ilgi alanları, çalışma yaklaşımı ve CV bağlantısı. |
| Projeler | Projelerin listelenmesi; konu veya teknolojiye göre filtreleme. |
| Proje detayları | Çözülen problem, projenin amacı, kişisel katkım, kullanılan teknolojiler, görseller, kaynak kod ve varsa canlı demo bağlantıları. |
| İletişim | E-posta ve GitHub, LinkedIn gibi kişisel bağlantılar. |

Gerçek isim, tanıtım metinleri, CV, sosyal bağlantılar ve proje içerikleri daha sonra sağlanacak. Taslaklardaki “Ad Soyad”, “Proje 01” ve “Proje 02” yer tutucudur.

Teknik notlar veya yazılar bölümü fikir olarak konuşuldu; ilk sürüm için kesin bir gereksinim olarak belirlenmedi.

## 3. Site içi arama

- Projeler ve yayımlanan site içeriği arasında arama yapılabilecek.
- Aramaya sayfanın üst bölümünden, kolay fark edilen bir alandan erişilecek.
- Yalnızca küçük bir büyüteç ikonuna veya sayfanın altındaki bir bölüme bağlı kalınmayacak.
- Kullanıcının istediği, aramayı çok büyütmek değil; yerleşim, etiket ve görünürlükle belirginleştirmek.
- Başlangıçta metin araması ve filtreleme düşünüldü. Anlamsal arama sonraki bir geliştirme olabilir; yöntem henüz seçilmedi.

## 4. Yapay zekâ asistanı

### Amaç ve davranış

- Kişisel tanıtım, yetenekler ve projeler hakkında soruları site içeriğine dayandırarak yanıtlamak.
- İlgili proje veya sayfa bağlantılarını göstermek.
- Kaynaklarda yeterli bilgi bulunmadığında bunu açıkça belirtmek.
- Site güncellendiğinde asistanın kullandığı bilgiyi de güncel tutmak.
- Proje bilgileri, arama ve asistan mümkün olduğunca ortak içerik kaynağından beslenecek.

Örnek sorular:

- “Hangi projeler üzerinde çalıştın?”
- “Hakkında bilgi verir misin?”
- “Belirli bir teknolojiyi kullandığın projeler hangileri?”

### Yerleşim

- Chatbot sağ tarafta, sayfa kaydırılırken de ekranda kalabilecek şekilde tasarlanacak.
- İçeriğin okunmasını ve proje bağlantılarının kullanılmasını engellemeyecek.
- Küçültülebilir panel, ikinci görsel taslakta gösterilen bir öneridir; panelin kesin boyutu ve açılış durumu henüz kararlaştırılmadı.
- Mobil yerleşim henüz tasarlanmadı.

### Çalıştırma ve bilgi kullanımı

- Proje kullanıcının kendi sunucusunda canlı çalışacak.
- Yapay zekâyı da sunucuda, istek başına harici model kullanım ücreti gerektirmeyen bir çözümle çalıştırma hedefi var.
- Modelin ücretsiz edinilebilmesi, sunucu kaynakları ve işletim maliyetinin sıfır olduğu anlamına gelmez. Model ve performans, sunucunun kapasitesine göre değerlendirilecek.
- Site içeriğini doğrudan modele verme ve ilgili içeriği arayıp modele aktarma (RAG) seçenekleri konuşuldu. Kesin yöntem seçilmedi.
- Modeli site bilgileriyle yeniden eğitme kararı alınmadı.
- Yanıt hızı ve doğruluğu için henüz ölçüm yapılmadı; yalnızca siteyi kaynak kullanma kuralı hatasızlık garantisi olarak görülmeyecek.

## 5. Etkileşimli projeler ve genişleme

- Site zamanla yeni araçlar, deneyler ve farklı teknolojilerle geliştirilmiş uygulamalar eklenebilecek şekilde düşünülecek.
- “Laboratuvar” adı ilk öneride kullanıldı, fakat kullanıcı bu adın neyi anlattığını açık bulmadı.
- Mevcut tasarım önerisi: Ayrı bir “Laboratuvar” bölümü yerine demoları ilgili proje altında “Canlı dene” bağlantısıyla sunmak.
- “Projeyi incele” proje açıklamasını; “Canlı dene” etkileşimli demoyu açacak. Canlı demo bağlantısı yalnızca demosu bulunan projelerde yer alacak.
- Bu alanın nihai adı ve yerleşimi tasarım araştırmasının ardından netleştirilecek.

## 6. Tasarım yönü ve ayrı tasarım belgesi

Tasarımın ayrıntıları, kullanıcı geri bildirimleri, hareket önerileri ve görsel referanslar [TASARIM.md](TASARIM.md) dosyasına taşındı. Güncel tasarım kaynağı bu ayrı belgedir.

- Ana referans: [Dennis Snellenberg](https://dennissnellenberg.com/).
- Beğenilen renk yönü: koyu mürekkep/füme, sıcak beyaz ve açık sarı-yeşil vurgular.
- Arama üstte belirgin, chatbot sağda kaydırma sırasında erişilebilir olacak.
- Farklı dillerde selamlaşma gösteren giriş animasyonu eklenecek.
- Portre arkasında/önünde akış ve fare etkileşimi denenecek; ayrıntılar henüz onaylanmadı.
- Dört statik görsel taslak [referanslar/tasarim/](referanslar/tasarim/) klasöründe saklanıyor. Hiçbiri nihai tasarım olarak onaylanmadı.

## 7. Temel kalite beklentileri

- Mobil uyumluluk.
- Okunaklı tipografi, açık içerik hiyerarşisi ve kolay gezinme.
- Erişilebilirlik.
- Hızlı açılış ve arama motorlarında düzgün görünme.
- Yeni proje ve özelliklerin eklenebilmesi.

## 8. İlk sürümün hedef kapsamı

1. Kişisel tanıtım ve hakkımda içeriği.
2. Proje listesi ve proje detayları.
3. İletişim bağlantıları.
4. Üst bölümden erişilen site içi arama.
5. Site içeriğine dayanan, sağda erişilebilir yapay zekâ asistanı.
6. Uygun projeler için canlı demo bağlantıları.

## 9. Teknik kararlar ve açık noktalar

- Backend dili kullanıcı kararıyla Rust olacak. Geliştirme AI destekli yapılacak; kullanıcının .NET mimari deneyimi referans alınacak.
- IM2026 örnek projesi salt okunur incelendi. Yeni mimari; iş kuralı, transaction, eşzamanlılık, telafi, test ve operasyon güvencelerini koruyacak şekilde önerildi.
- Next.js/React/TypeScript, Axum/Tokio/Tower ve PostgreSQL/SQLx önerilen başlangıç teknolojileridir. Kesin karar/öneri ayrımı ve teknik ayrıntılar [TEKNIK_MIMARI.md](TEKNIK_MIMARI.md) dosyasında yer alır.
- Sunucuda canlı çalışma kararı var; sunucunun donanımı ve kapasitesi henüz paylaşılmadı.
- Nihai görsel düzen, asistanın davranışı ve mobil tasarım henüz netleşmedi.
- İçerik düzenleme yöntemi, yönetim paneli ve kullanıcı hesabı gereksinimleri henüz belirlenmedi.
- AI modeli/runtime'ı ve dağıtım araçları sunucu ve kalite denemesiyle seçilecek.
- Kapsam, tasarım ve teknik mimari belgeleri hazırlandı. Uygulama geliştirmeye geçilmedi.

# Faz 2A — Content Export ve Türkçe Arama Motoru

**Amaç:** Astro'nun ürettiği içerik sözleşmesinden (export v1), Rust ile Türkçeye uygun bir arama indeksi ve bundle üretmek; bunu komut satırından sorgulanabilir ve test edilebilir hâle getirmek.
**Önkoşul:** Faz 1 çıkış kapısı (İP-1.13 ertelenmiş olabilir).
**Referans:** V2 §5 (sahiplik, sözleşme yönleri), §6.4 (export v1), §8 (Rust), §9 (arama), §17.
**Kapsam dışı:** HTTP API (Faz 2B), asistan (Faz 4), anlamsal arama/embedding (V2 §9.5; yalnız gerekirse, ayrı karar).

Bu fazda async kod yoktur. Rust odağı: serde, tipli hatalar, sahiplik, iterator'lar, trait uygulama (Tantivy filtresi).

---

## İP-2A.1 — Content export v1 (Astro)

**Amaç:** Rust tarafının içerikle ilgili tek girdisini üretmek. Sözleşmenin sahibi Astro'dur (D-08).

**Yapılacaklar:**

1. Tipler `web/src/lib/export/types.ts`:

   ```ts
   export interface ExportV1 {
     schemaVersion: 1;
     release: { gitCommit: string };                             // zaman damgası yok: aynı commit → aynı export
     site: {
       name: string; headline: string; shortBio: string; skills: string[];
       links: { email: string; github: string | null; linkedin: string | null; cv: string | null };
     };
     tags: ExportTag[];                                          // id'ye göre sıralı
     items: ExportItem[];                                        // id'ye göre sıralı
   }
   export interface ExportTag { id: string; label: string; aliases: string[]; group: string }
   export interface ExportItem {
     id: string;                       // "project/<slug>" | "page/about"
     kind: "project" | "page";
     url: string;                      // kanonik yol, "/" ile başlar
     title: string; summary: string;
     role: string | null; status: string | null; year: number | null;
     tags: string[];                   // etiket id'leri
     featured: boolean; order: number;
     links: { repo: string | null; demo: string | null };
     sections: ExportSection[];
   }
   export interface ExportSection {
     anchor: string;                   // giriş bölümünde ""
     heading: string;
     url: string;                      // hazır bağlantı: item.url veya item.url + "#" + anchor (Rust URL kurmaz)
     text: string;
   }
   ```

   Opsiyonel alanlar her zaman bulunur ve değer yoksa `null` olur (alan atlanmaz).
2. Bölüm çıkarıcı `web/src/lib/export/sections.ts` (saf fonksiyon, testli):
   - Girdi: ham gövde (`entry.body`), dosya türü (md/mdx), Astro'nun `render(entry)` ile verdiği `headings`.
   - unified + remark-parse (+ remark-gfm; MDX için remark-mdx) ile AST.
   - Bölme yalnız `h2` ve `h3`'te. İlk başlıktan önceki metin, `anchor: ""` ve `heading: <öğe başlığı>` ile giriş bölümüdür.
   - Metin dönüşümü: paragraf → metin; liste öğesi → `- metin` satırı; tablo → satır başına hücreler ` | ` ile; alıntı → metin; görsel → `Görsel: <alt>` (alt boşsa atlanır). Kod blokları, ham HTML, MDX JSX/ESM/ifade düğümleri **atlanır**. Bloklar `\n\n` ile birleşir; satır içi boşluklar sadeleşir.
   - Anchor'lar: Astro `headings` listesinden derinliği 2–3 olanlar sırayla eşlenir. Sayı veya başlık metni uyuşmazsa dosya yolunu içeren bir hatayla derleme durur.
   - Metni boş kalan bölüm atlanır.
3. Kurucu `web/src/lib/export/build.ts`: `content.ts` yardımcılarını kullanır (draft hariç, yayımlanmış içerik). Hakkımda sayfası `page/about` öğesidir: `url: "/hakkimda"`, `title: "Hakkımda"`, `summary: profile.shortBio`, bölümler profil gövdesinden. Diziler id'ye göre sıralanır.
4. Endpoint `web/src/pages/internal/content-export.json.ts` (prerender) → `dist/internal/content-export.json`.
5. `just export`: web derlemesi → dosyayı `out/export/content.v1.json`'a kopyalar. Sitemap ve paket zaten `/internal`'ı dışlar (İP-1.6, İP-1.8); Caddy de 404 döner (İP-1.7). `headers.spec.ts`'e gerçek yol eklenir: `/internal/content-export.json` → 404 (yerel testte dosya `dist`'te bulunduğu hâlde).

**Örnek çıktı (kısaltılmış):**

```json
{
  "schemaVersion": 1,
  "release": { "gitCommit": "a1b2c3…" },
  "site": { "name": "Ad Soyad", "headline": "…", "shortBio": "…", "skills": ["C#", "Rust"],
            "links": { "email": "ad@example.com", "github": null, "linkedin": null, "cv": null } },
  "tags": [{ "id": "csharp", "label": "C#", "aliases": ["c#", "csharp", "c sharp"], "group": "dil" }],
  "items": [{
    "id": "project/ornek-proje-1", "kind": "project", "url": "/projeler/ornek-proje-1",
    "title": "Örnek Proje 1", "summary": "…", "role": "…", "status": "tamamlandi", "year": 2025,
    "tags": ["csharp", "dotnet"], "featured": true, "order": 1,
    "links": { "repo": null, "demo": null },
    "sections": [
      { "anchor": "", "heading": "Örnek Proje 1", "url": "/projeler/ornek-proje-1", "text": "Giriş paragrafı…" },
      { "anchor": "problem", "heading": "Problem", "url": "/projeler/ornek-proje-1#problem", "text": "…" }
    ]
  }]
}
```

**Testler (Vitest):** Türkçe başlıklar, liste/tablo/kod/MDX bileşeni içeren örnek gövdeler; kod ve JSX'in atlanması; anchor eşleme hatası; boş bölüm; sıralama. Fixture'lar `web/tests/fixtures/export/`.

**Bitti sayılır:**

- [ ] `just export` → `out/export/content.v1.json` üretiliyor; `dist` paketinde `internal/` yok.
- [ ] Çıktıdaki her bölüm URL'si (`url#anchor`) sitede gerçekten var (İP-1.4 bağlantı denetimi anchor'ları da kontrol ediyor).

---

## İP-2A.2 — Rust workspace ve `contract` crate'i

**Yapılacaklar:**

1. İP-1.8'deki workspace'e yeni üyeler: `crates/contract`, `crates/search`, `crates/indexer`, `crates/eval` (son ikisi binary; `eval` İP-2A.8'de doldurulur, şimdilik boş `main`).
2. `contract` crate'i:
   - `export` modülü: İP-2A.1 tiplerinin Rust karşılıkları. `#[serde(rename_all = "camelCase", deny_unknown_fields)]`. `schema_version != 1` → hata.
   - `ids` modülü: `ItemId`, `TagId`, `ChunkId`, `SourceLabel` newtype'ları; kurucular doğrular (`ItemId`: `project/<slug>` veya `page/<ad>`; `TagId`: `^[a-z0-9]+(-[a-z0-9]+)*$`).
   - `CONTRACT_VERSION: u32 = 1` (API sözleşmesi; İP-2B.3 kullanır).
   - `validate(export) -> Result<(), Vec<ExportIssue>>`: id tekilliği, URL'lerin `/` ile başlaması, bölüm URL'sinin öğe URL'siyle başlaması, öğe etiketlerinin `tags` içinde bulunması, öğe başına anchor tekilliği, boş başlık olmaması.
3. Test fixture'ı `backend/fixtures/export-v1.json`: gerçek içerikten bağımsız, elle yazılmış, Türkçe. Arama testleri bunu kullanır. En az:
   - 8 proje + `page/about`.
   - Etiketler: `csharp`, `dotnet`, `aspnetcore`, `rust`, `react`, `typescript`, `nodejs`, `cpp`, `postgresql`, `docker`.
   - Metinlerde: "öğrenci", "öğrencilerin", "geliştirdiğim", "İstanbul", "ılık", "ISITMA", ekli biçimler ("projelerimde", "uygulamasında"), teknoloji adları (".NET 8", "C#", "C++", "Node.js", "ASP.NET Core"), kesme işaretli ekler ("C#'ta", "React'le").
   - Bilerek benzer adlı iki proje (ör. "Öğrenci Takip" ve "Öğretmen Takip").

**Testler:** fixture ayrıştırılıyor; bilinmeyen alan → hata; geçersiz id → hata; `validate` her kuralı yakalıyor.

**Bitti sayılır:**

- [ ] `cargo test -p contract` geçiyor.
- [ ] CI: gerçek export (`out/export/content.v1.json`) da `contract` ile ayrıştırılabiliyor (İP-2A.9'da otomatikleşir; şimdilik elle).

---

## İP-2A.3 — Türkçe metin işleme ve etiket eşleştirici

**Amaç:** Türkçe arama kalitesinin temeli. Referans: V2 §9.2, §9.3.

**Yapılacaklar (`search` crate):**

1. `text` modülü (saf fonksiyonlar):
   - `nfc(s)`: Unicode NFC.
   - `turkish_lowercase(s)`: önce `I → ı`, `İ → i`, ayrıca `i` + birleşen nokta (U+0307) dizisi → `i`; sonra Unicode küçültme.
   - `fold_ascii(s)`: `ç→c, ğ→g, ı→i, ö→o, ş→s, ü→u` (+ `â→a, î→i, û→u`); `turkish_lowercase` sonrasında uygulanır.
2. Tantivy filtresi `TurkishLowerCaser`: Tantivy'nin `LowerCaser`'ı örnek alınarak yazılır; `turkish_lowercase` kuralını uygular. Saklanan özgün metin değişmez (vurgu konumları doğru kalır).
3. Analizörler (`analyzers::register(&index)`; indeks her açıldığında çağrılır, çünkü Tantivy özel tokenizer'ları saklamaz):

   | Ad | Zincir |
   | --- | --- |
   | `tr_stem` | SimpleTokenizer → RemoveLongFilter(40) → TurkishLowerCaser → Stemmer(Turkish) |
   | `tr_fold` | SimpleTokenizer → RemoveLongFilter(40) → TurkishLowerCaser → AsciiFoldingFilter |
   | `tr_prefix` | NgramTokenizer(2, 15, yalnız önek) → TurkishLowerCaser → AsciiFoldingFilter |
   | `raw` | RawTokenizer (etiket id'leri) |

4. `tags::TagMatcher`:
   - Girdi: export'taki etiketler. Her etiket için anahtarlar: `label`, `id`, `aliases`. Anahtar normalizasyonu: `fold_ascii(turkish_lowercase(nfc(x)))`, boşluklar tek boşluk.
   - Sorgu, **yalnız boşluktan** bölünür (`.`, `#`, `+` korunur). Her parçanın sonundaki kesme işaretli ek atılır (`'` veya `’` ve sonrası: "C#'ta" → "c#", ".NET'le" → ".net").
   - En uzun eşleşme: parça dizileri üzerinde, çok kelimeli eşanlamlılar önce ("asp.net core" iki parça).
   - Çıktı: eşleşen `TagId`'ler (tekil, sıralı) + eşleşen parçalar çıkarılmış kalan metin.

**Testler (tablo tabanlı):**

| Girdi | Beklenen |
| --- | --- |
| `turkish_lowercase("İSTANBUL ILIK")` | `istanbul ılık` |
| `turkish_lowercase("ÖĞRENCİ")` | `öğrenci` |
| `fold_ascii("öğrenci ışık")` | `ogrenci isik` |
| "C# projelerin" | etiket `csharp`, kalan "projelerin" |
| ".NET'le yaptığın" | `dotnet`, kalan "yaptığın" |
| "ASP.NET Core ile" | `aspnetcore` (dotnet değil), kalan "ile" |
| "c++ ve node.js" | `cpp`, `nodejs`, kalan "ve" |
| "Rust’ta" (tipografik kesme) | `rust` |
| "react" | `react` |
| "reaktör" | etiket yok |

**Bitti sayılır:**

- [ ] Testler geçiyor; `TurkishLowerCaser` vurgu konumlarını bozmuyor (özgün metinde "İ" içeren bir parça doğru vurgulanıyor).

---

## İP-2A.4 — Bölme ve bundle modeli

**Yapılacaklar:**

1. `chunk` modülü:
   - `estimate_tokens(text) = ceil(char_count / 3.0)` (Türkçe için temkinli yaklaşım; sabit `CHARS_PER_TOKEN_EST`).
   - Bölüm ≤ 350 tahmini token → tek parça. Değilse paragraf sınırlarından açgözlü bölme (≤ 350); tek paragraf büyükse cümle sınırından (`. `, `? `, `! `), o da yetmezse karakter sınırından bölünür.
   - `ChunkId` = `blake3(item_id ␟ anchor ␟ ordinal)` ilk 16 onaltılık karakter.
   - `context_prefix`: proje için `Proje: {title} — Bölüm: {heading}`, sayfa için `Sayfa: {title} — Bölüm: {heading}`. Önek indekslenmez; asistan bağlamında kullanılır.
2. `bundle` modülü — dizin düzeni:

   ```text
   bundle/
     manifest.json   { schemaVersion: 1, release: {gitCommit}, contentHash: "blake3:<hex>",
                       indexerVersion, analyzerVersion: 1, counts: {items, chunks}, totalEstTokens }
     items.json      [{ id, kind, url, title, summary, role, status, year, tags, featured, order, links }]
     tags.json       [{ id, label, aliases, group }]
     chunks.json     [{ id, itemId, anchor, heading, url, ordinal, contextPrefix, text, estTokens }]
     search/         Tantivy indeksi
   ```

   - `contentHash`, export dosyasının baytlarından hesaplanır. İndeksleyici zaman damgası eklemez; aynı export → aynı `manifest.json`, `items.json`, `tags.json`, `chunks.json` (bayt bayt). Tantivy dosyalarının baytları farklı olabilir; sorgu sonuçları aynıdır.
   - `Bundle::open(dir)`: şema sürümünü, dosyaların varlığını ve sayıları doğrular, analizörleri kaydeder, indeksi salt okunur açar, öğe ve parça haritalarını belleğe alır.

**Testler:** bölme kuralları (sınırlar, uzun paragraf, uzun cümle); kimlik kararlılığı; aynı fixture'dan iki derlemede JSON dosyaları bayt bayt aynı; bozuk/eksik bundle → anlaşılır hata.

**Bitti sayılır:**

- [ ] Testler geçiyor.

---

## İP-2A.5 — Tantivy indeks şeması ve kurucu

**Şema:**

| Alan | Tür / analizör | Saklanır | Belge türü |
| --- | --- | --- | --- |
| `doc_type` | raw (`item` / `chunk`) | evet | ikisi |
| `item_id` | raw | evet | ikisi |
| `chunk_id` | raw | evet | chunk |
| `title_stem` | `tr_stem` | hayır | item |
| `title_fold` | `tr_fold` | hayır | item |
| `title_prefix` | `tr_prefix` (başlığın her kelimesi ayrı değer) | hayır | item |
| `tags` | raw (çok değerli) | evet | ikisi |
| `summary_stem`, `summary_fold` | `tr_stem`, `tr_fold` | `summary_stem` evet | item |
| `heading_stem` | `tr_stem` | hayır | chunk |
| `text_stem` | `tr_stem` | hayır (metin `chunks.json`'da) | chunk |
| `text_fold` | `tr_fold` | hayır | chunk |

**Kurucu:** `IndexBuilder::build(&ExportV1, &[Chunk], dir)`: tek yazar iş parçacığı, belgeler sıralı eklenir, tek commit, birleştirme beklenir.

**Not:** Tantivy `Index::open` özel analizörlerin kayıtlı olup olmadığını denetlemez; hata ancak kullanımda çıkar. Bu yüzden `Bundle::open`, analizörleri kaydettikten sonra her metin alanı için `index.tokenizer_for_field(alan)` çağırır ve başarısızlığı açılış hatasına çevirir.

**Testler:** fixture'dan indeks kurulup yeniden açılıyor; belge sayıları (öğe + parça) doğru; analizör kaydı atlanmış bir açılış `tokenizer_for_field` denetiminde hata veriyor.

**Bitti sayılır:**

- [ ] Testler geçiyor.

---

## İP-2A.6 — Sorgu motoru

**Amaç:** V2 §9.3'teki sorgu hattı. Hem arama kutusu (öğe sonuçları) hem asistan (parça sonuçları) bunu kullanır.

**API:**

```rust
pub struct Engine { /* Bundle, Index, IndexReader, Fields, TagMatcher */ }

impl Engine {
    pub fn open(bundle_dir: &Path) -> Result<Self, SearchError>;
    pub fn search(&self, req: &SearchRequest) -> Result<SearchResults, SearchError>;      // öğe düzeyi
    pub fn search_chunks(&self, query: &str, limit: usize) -> Result<Vec<ChunkHit>, SearchError>; // asistan için
    pub fn tag_matcher(&self) -> &TagMatcher;
}

pub struct SearchRequest { pub q: String, pub tags: Vec<TagId>, pub limit: usize }
```

**Doğrulama:** `q` kırpıldıktan sonra ≤ 200 karakter; `tags` ≤ 5; `limit` 1–20 (varsayılan 8). `q` boş ve `tags` boş → hata. "Sorgu boşlukla bitiyor mu" bilgisi kırpmadan **önce** ham girdiden alınır (önek eşleşmesi için). `search_chunks` bu sınıra tabi değildir: kendi sınırı 1200 karakterdir ve fazlası hata değil, sondan kırpmadır (asistanın geri getirme sorgusu mesaj + önceki mesajdır).

**Hat:**

1. NFC → `TagMatcher` → eşleşen etiketler + kalan metin.
2. Kalan metin `tr_stem` ve `tr_fold` analizörleriyle terimlere ayrılır (`index.tokenizers()` üzerinden).
3. `BooleanQuery` (Should cümleleri; en az biri eşleşmeli), başlangıç ağırlıkları `weights` modülünde adlandırılmış sabitler:

   | Cümle | Ağırlık |
   | --- | --- |
   | `title_stem`, `title_fold` terimleri | 3.0 |
   | Sorgudan eşleşen etiketler (`tags` alanı) | 2.5 |
   | `summary_stem`, `summary_fold`, `heading_stem` | 1.5 |
   | `text_stem`, `text_fold` | 1.0 |
   | Son kelime (ham sorgu boşlukla bitmiyorsa, ≥ 2 karakter) → `title_prefix` üzerinde **tek** `TermQuery` | 1.0 |
   | ≥ 5 karakterli fold terimleri için `title_fold` üzerinde `FuzzyTermQuery::new(terim, 1, true)` (yer değiştirme dahil) | 0.5 |

   Önek terimi: son kelime `turkish_lowercase` + `fold_ascii` uygulanır, 15 karakterde kesilir ve olduğu gibi tek terim olur. `tr_prefix` analizöründen **geçirilmez** (geçirilirse bütün 2–15 harfli önekleri üretir ve iki harfi ortak her başlık eşleşir).

4. İstekteki `tags` parametresi (kullanıcının seçtiği filtre) **Must** cümlesidir (her biri AND).
5. Kalan metin boş ama etiket eşleşmişse: o etiketlere sahip öğeler, `featured`/`order` sırasıyla.
6. İlk 200 belge alınır, `item_id`'ye göre gruplanır: öğe puanı = en yüksek belge puanı; en iyi parça vurgu için seçilir.
7. Sıralama: puan (azalan) → `featured` → `order` → `slug`. `limit` uygulanır.
8. Vurgu: `text_stem` ve `text_fold` için birer `SnippetGenerator` (≤ 160 karakter) oluşturulur; ikisine de en iyi parçanın metni (`chunks.json`'dan) `.snippet(metin)` ile verilir; vurgu içeren sonuç kullanılır (şapkasız sorgular yalnız `text_fold`'dan vurgu alır). Sonuç `[{ text, highlighted }]` parçalarıdır. Parça eşleşmesi yoksa özet, vurgusuz.
9. `search_chunks`: aynı sorgu hattı, yalnız `doc_type = chunk`; parça puanına göre ilk `limit` (gruplama yok); her sonuç `ChunkId`, `ItemId`, puan.

**Kural:** Kullanıcı girdisi asla `QueryParser`'a verilmez.

**Sonuç tipi:** `SearchResults { items: Vec<SearchHit>, matched_tags: Vec<TagId>, took: Duration }`; `SearchHit { item_id, section: Option<{ heading, url }>, snippet: Vec<Fragment>, score }`. Bölüm URL'si parçanın `url` alanından gelir. Puan yalnız iç kullanım içindir; API'ye çıkmaz.

**Testler:** İP-2A.8'deki altın testlere ek olarak birim testleri: doğrulama hataları; Must filtresi; yalnız etiket sorgusu; gruplama ve kararlı sıralama; vurgu parçaları.

**Bitti sayılır:**

- [ ] Testler geçiyor; fixture üzerinde tek sorgu < 5 ms (yerel ölçüm, raporda belirtilir).

---

## İP-2A.7 — `site-indexer` CLI

**Komutlar** (`indexer` crate, clap; çıktılar Türkçe):

- `site-indexer check --export <file>`: ayrıştırma + `contract::validate`; sorunları listeler; çıkış 1.
- `site-indexer build --export <file> --out <dir>`: check → bölme → indeks → JSON dosyaları → manifest. Önce `<dir>.tmp`'ye yazar, sonra yer değiştirir. Özet: öğe, parça, toplam tahmini token, en büyük parça.
- `site-indexer query --bundle <dir> [--tags a,b] [--limit n] [--chunks] [--explain] "<sorgu>"`: sonuçları tablo hâlinde yazar. `--explain`: eşleşen etiketler, alan başına terimler, puanlar. `--chunks`: parça sonuçları (asistanın göreceği).
- `site-indexer stats --bundle <dir>`.

**`justfile`:** `index` = `export` + `site-indexer build --export out/export/content.v1.json --out out/bundle`; `query q` kısayolu.

**Windows notu:** Çalışan bir `site-api` indeks dosyalarını bellek eşlemeli tutar; Windows'ta bu sırada `out/bundle` değiştirilemez. `site-indexer build` hedef kullanılıyorsa "Bundle kullanımda; önce API'yi durdurun" hatasıyla durur. `just dev` (İP-2B.4) indeksi API'yi başlatmadan önce kurar.

**Bitti sayılır:**

- [ ] `just index` ve `just query "c# projeleri"` çalışıyor; `--explain` çıktısı anlaşılır.

---

## İP-2A.8 — Altın testler ve `site-eval search`

**Yapılacaklar:**

1. `backend/fixtures/search-cases.toml`: fixture üzerinde en az 20 sorgu. Kapsam:
   - Şapkasız yazım ("ogrenci", "gelistirdigim"), büyük harf ("ÖĞRENCİ", "İSTANBUL"), ekler ("projelerimde"), teknoloji adları (".NET", "dotnet", "C#", "c sharp", "C++", "Node.js", "ASP.NET Core"), yazım hatası ("ogrneci" → fuzzy başlıkta), benzer adlar ("öğrenci takip" yalnız doğru projeyi ilk sıraya almalı), yalnız etiket, çok kelimeli sorgu, sonuç beklenmeyen sorgu ("kuantum fiziği").
2. `backend/crates/search/tests/golden.rs`: fixture bundle'ı bir kez kurar (`OnceLock` + geçici dizin), her sorgu için sıralı öğe id'leri ve eşleşen etiketler `insta` YAML anlık görüntüsüne yazılır. Bilinçli değişiklikte anlık görüntü `cargo insta review` ile güncellenir; rapor bunu belirtir.
3. `eval` crate'i → `site-eval search --bundle <dir> --cases <toml>`:

   ```toml
   [[case]]
   id = "dotnet-projeleri"
   q = ".NET"
   expect_top3 = ["project/ornek-proje-1"]   # bu öğeler ilk 3'te olmalı

   [[case]]
   id = "olmayan-konu"
   q = "kuantum fiziği"
   expect_none = true
   ```

   Rapor: geçen/kalan, kalan vakalar için ilk 5 sonuç. Kalan varsa çıkış 1.
4. `eval/search.toml`: gerçek içerik için. Şimdilik yer tutucu içerikle 3–5 vaka ve dosyanın başında "gerçek içerik gelince güncelle" yorumu.
5. `just eval` → `site-eval search --bundle out/bundle --cases eval/search.toml`.

**Bitti sayılır:**

- [ ] Altın testler ve `just eval` geçiyor.
- [ ] Rapor, ağırlıklarla ilgili gözlemleri içeriyor (hangi vaka hangi ağırlığa duyarlı).

---

## İP-2A.9 — CI ve sürüm paketine bundle

**Yapılacaklar:**

1. CI `web` işi export'u artefakt olarak yükler; `rust` işi indirir ve:
   - `site-indexer check` (gerçek export ile sözleşme doğrulaması — D-08),
   - `site-indexer build`,
   - `site-eval search --cases eval/search.toml`.
2. `just release`, İP-1.10 adım 4'ü doldurur: **aynı derlemenin** `dist`'inden export alınır, bundle kurulur ve `--bundle out/bundle` ile paketlenir; `release.json` → `components.bundle = true`. Paketleme, bundle'ın `release.gitCommit` değeri commit'le aynı değilse reddeder (İP-1.8).
3. Deploy iş akışı (İP-1.13 varsa): `build-rust` işi `site-indexer`'ı da derler; `package` işi export'tan bundle kurar. İP-1.13 henüz yapılmadıysa bu madde o İP'ye eklenir.

**Bitti sayılır:**

- [ ] Yerelde: Astro tarafında export şemasını bozan bir değişiklik (ör. bir alan adını değiştirmek) `site-indexer check`'i düşürüyor; değişiklik geri alınıyor.
- [ ] **Kullanıcı kontrolü:** aynı deneme bir PR'da CI'ın `rust` işini düşürüyor.

---

## Faz 2A çıkış kapısı

- [ ] `just index`, `just query`, `just eval` çalışıyor.
- [ ] Altın testlerde V2 §17.2 senaryo 8 (Türkçe varyasyonlar ve teknoloji adları) kapsanıyor.
- [ ] CI yeşil; sürüm paketi bundle içeriyor.
- [ ] Elle kontrol: kullanıcı `just query` ile 5–10 kendi sorgusunu denedi ve sonuçlar makul.

**Faz 2B'ye devir:** `search::Engine` API'si ve `contract` crate'i hazır. `Engine::search_chunks` Faz 4'te kullanılacak.

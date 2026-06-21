# Yazgi — Proje Değerlendirme Raporu

> Tarih: 2026-06-21
> Kaynak: `README.md`, `dev/plan.md`, `dev/vision.md` dosyalarının ve repo durumunun okunması.

## Repo Durumu

Repo şu an **sadece plan**: kod yok. Yalnızca şu dosyalar mevcut:

- `README.md` — kısa proje tanıtımı
- `dev/plan.md` — ilk tek-oyunculu metin macerası iskeleti
- `dev/vision.md` — genişletilmiş çok-oyunculu AI-DM vizyonu (öncelikli doküman)

## Genel Değerlendirme

Fikir ve mimari olarak **olgun ve net düşünülmüş** bir proje. `vision.md` özellikle iyi
yazılmış: kararlar tablolaştırılmış, prensipler açık, MVP kapsamı "dahil/hariç" diye
ayrılmış. Çoğu hobi projesinin atladığı şeyleri (sağlayıcı bağımsızlığı, hafıza-önce
tasarım, deterministik kural hakemliği) baştan düşünmüş. İki doküman arasındaki öncelik
ilişkisi de açıkça tanımlanmış (`vision.md` önceliklidir). Bu zihinsel netlik en büyük artı.

## En Güçlü Kararlar

- **Yapılandırılmış state + tool-use + backend doğrulama.** "LLM'i tamamen serbest
  bırakma" içgüdüsü doğru. HP/envanter/konum DB'de katı, atmosfer anlatıda — bu ayrım
  projenin kalbi ve doğru yere konmuş.
- **Zarın backend'de atılması.** AI `roll_check` çağırır, sonucu backend deterministik
  üretir. Adil, denetlenebilir, LLM'in "kayırmasını" engeller. Çok yerinde.
- **Evrenden bağımsız çekirdek + ruleset eklentisi.** D&D mantığını çekirdeğe gömmemek,
  projeyi tek bir oyuna kilitlemiyor.
- **Provider soyutlaması + yerel embedding.** Embedding'i Ollama'ya, kritik DM kararını
  Claude'a vermek maliyet/kalite dengesi açısından akıllıca.

## Endişeler / Riskler

1. **Kapsam çok hızlı büyümüş.** `plan.md` tek-oyunculu Zork iskeleti; `vision.md`
   çok-oyunculu + AI-DM + karakter sistemi + çoklu evren + host gözetimi + async kuyruk.
   İkincisi tek bir MVP için **büyük** (şema 8 tabloya çıkmış). Risk: hiçbiri bitmeden
   her şeye başlanması. Vizyon hedef olarak kalsın, ama ilk çalışan ürün `plan.md`'ye
   daha yakın olmalı.

2. **En zor şey (DM kalitesi/tutarlılık) en az tanımlı olan.** "MVP ilk odak: AI DM
   kalitesi" denmiş ama dokümanlar altyapıya (tablolar, provider, routes) odaklanıyor.
   Asıl değer ve asıl zorluk: sistem promptu, hafıza derleme stratejisi, ne zaman
   özetleneceği, retrieval'ın gerçekten işe yarayıp yaramadığı. Bunlar `memory.ts`
   satırında bir cümleyle geçiştirilmiş. Proje burada batar ya da yüzer.

3. **pgvector + yerel Ollama embedding'i MVP için erken karmaşıklık olabilir.** "Son N
   olay + özet" çoğu kısa kampanya için zaten yeterli. Semantik retrieval'ı, basit hafıza
   tutarsızlık yaşatana kadar ertelemek MVP'yi hızlandırır.

4. **İki dokümanın yarattığı belirsizlik / çelişkiler.**
   - `plan.md` "çift istemci (terminal + React)" diyor, `vision.md` "önce CLI, web sonra"
     diyor.
   - `plan.md` embedding boyutunu `1536` (OpenAI), `vision.md` `768` (nomic) yazmış.
   - Çelişkiler küçük ama kod yazmaya başlayınca kafa karıştırır. `plan.md`'yi vizyona
     göre güncellemek veya arşivlemek gerekiyor.

5. **Çok-oyunculu + async senkronizasyonu sessiz bir karmaşıklık kaynağı.** "DM ne zaman
   biriken aksiyonları işler" sorusu (Açık Sorular'da kabul edilmiş) aslında çekirdek
   tasarım kararı — MVP'de en basitiyle (host bir komutla turu kapatır) çözülmeli, yoksa
   zamanlayıcı/bildirim altyapısına kayar.

## Öneri

Vizyonu koruyup MVP'yi **bilinçli olarak küçültmek**:

- **Önce tek oyuncu + `generic` ruleset + CLI** ile DM döngüsünü çalışır hale getir
  (state + tool-use + doğrulama). Çok-oyunculuyu şema seviyesinde hazırla ama davranışı
  sonraya bırak.
- **Hafıza = "son N olay + özet" ile başla**, pgvector'ü sonraki adıma al.
- **İlk gerçek iş DM prompt'u ve tool-use döngüsü olsun**, route/şema değil — değerin
  olduğu yere önce dokun.
- **`plan.md`'yi `vision.md` ile uyumla** (embedding boyutu, istemci stratejisi) ki tek
  bir doğru kaynak olsun.

## Özet

Tasarım kalitesi yüksek, yön doğru; tek gerçek risk **kapsam disiplini** — vizyonu hedef
tutup MVP'yi acımasızca küçük başlatmak.

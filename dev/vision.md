# Yazgi — Vizyon & Mimari Tasarımı: Çok Oyunculu AI-DM FRP Platformu

> Bu doküman, 2026-06-21 tarihli soru-cevap görüşmesinin çıktısıdır.
> `dev/plan.md` mevcut tek-oyunculu metin macerası iskeletini tanımlar; bu doküman
> projenin genişletilmiş hedefini (çok oyunculu, AI Dungeon Master, karakter sistemi,
> çoklu evren) ve onu gerçekleştirecek mimariyi tanımlar. Çelişki olduğunda **bu doküman
> önceliklidir** ve `plan.md` zamanla buna doğru evrilecektir.

---

## 1. Vizyon

Yazgı, arkadaş gruplarının birlikte oynayabileceği, **AI'nın Oyun Yöneticisi (Dungeon Master / DM)**
olarak davrandığı, açık kaynaklı bir **çok oyunculu FRP (Fantezi Rol Yapma) oyun platformudur.**

Sadece Dungeons & Dragons değil; **Cyberpunk, Sci-Fi ve başka evrenler** de "dünya tanımı"
(world definition) olarak eklenebilir. Sistem, evrenden bağımsız bir çekirdek üzerinde,
her evrenin kendi kurallarını eklenti olarak getirdiği genişletilebilir bir yapıdır.

---

## 2. Görüşmede Alınan Kararlar (Özet)

| Konu | Karar |
|------|-------|
| **Oyun akışı** | **Sıra tabanlı (turn-based) VE asenkron (play-by-post)** — oda kurulurken seçilir. İkisi de ortak "aksiyon kuyruğu" altyapısını paylaşır. Gerçek zamanlı WebSocket MVP kapsamı dışında. |
| **DM rolü** | **AI DM + insan gözetimi.** AI yönetir; bir host (oda sahibi) gerektiğinde araya girip kararı düzeltebilir, sahneyi yönlendirebilir. |
| **Kural sistemi** | **Genel kural motoru + evren başına ruleset eklentisi.** Çekirdek mekanik (özellik, kontrol/check, zar) + her evrenin kendi `ruleset` modülü. |
| **MVP ilk odak** | **AI DM kalitesi.** Önce DM'i terminalde olgunlaştır. |
| **DM kritik yetenekleri** | **(1) Tutarlılık & hafıza, (2) Sürükleyici anlatı.** |
| **Durum (state) yönetimi** | **Yapılandırılmış state + AI tool-use.** Karakter/envanter/HP gerçek DB kaydı; AI tool-call ile okur/değiştirir; backend doğrular. |
| **Teknoloji** | **Node.js / TypeScript + Express + PostgreSQL (pgvector) + Anthropic SDK.** Mevcut yığın korunuyor. |
| **LLM stratejisi** | **Sağlayıcı-bağımsız (provider-agnostic) arayüz, varsayılan Claude.** Config ile Ollama/Gemma'ya geçilebilir (geliştirme/test'te yerel, gerçek oyunda Claude). |
| **Embedding** | **Yerel embedding (Ollama, ör. `nomic-embed-text`)** → pgvector. Bedava, gizli, hafıza/retrieval için yeterli. |
| **İlk arayüz** | **Terminal / CLI önce.** DM kalitesini burada olgunlaştır; web/Discord sonra. |

---

## 3. Çekirdek Tasarım Prensipleri

1. **AI yönetir, insan gözetir.** AI DM sahneleri kurar, NPC'leri canlandırır, kuralları
   uygular; host (oda sahibi) son sözü söyleyebilir ve gerektiğinde düzeltir.
2. **Yapılandırılmış gerçek + serbest anlatı.** "Sayısal/kritik" veriler (HP, envanter,
   konum, karakter özellikleri) veritabanında katı tutulur; atmosfer, NPC ruh hali,
   ilişkiler anlatıda yaşar. Tutarlılık için tool-use doğrulaması zorunludur — LLM'in
   tamamen serbest bırakılması state tutarsızlığına yol açar.
3. **Evrenden bağımsız çekirdek.** Hiçbir D&D'ye özel mantık çekirdeğe gömülmez; her şey
   ruleset eklentisi ve world definition üzerinden gelir.
4. **Sağlayıcı kilidi yok.** Tüm LLM çağrıları tek bir soyutlama arkasından geçer.
5. **Hafıza birinci sınıf vatandaştır.** Uzun kampanyaların ölüm sebebi tutarsızlıktır;
   bu yüzden hafıza/retrieval baştan tasarlanır.

---

## 4. Mimari Genel Bakış

```
Yazgi/
  package.json                 (npm workspaces kökü)
  docker-compose.yml           (postgres + pgvector + opsiyonel ollama servisi)
  .env.example
  server/
    src/
      server.ts
      config.ts                (LLM sağlayıcı seçimi, env okuma)
      db/
        client.ts
        schema.sql             (worlds, campaigns, players, characters, sessions,
                                 turns/actions, events+embedding, dm_decisions)
      llm/                     # <-- SAĞLAYICI-BAĞIMSIZ KATMAN
        provider.ts            (LLMProvider arayüzü: complete(), completeWithTools(), embed())
        anthropic.ts           (Claude implementasyonu — varsayılan)
        ollama.ts              (Ollama/Gemma implementasyonu)
        index.ts               (config'e göre doğru provider'ı seçen fabrika)
      agent/
        dungeonMaster.ts       (DM çekirdeği: sistem promptu, tool-use döngüsü, sahne yönetimi)
        memory.ts              (bağlam derleme: world + state + son olaylar + pgvector retrieval + özetleme)
        tools.ts               (state-mutation tool şemaları)
        oversight.ts           (host müdahalesi: kararı onayla/düzelt/reddet)
      rules/                   # <-- KURAL MOTORU + EKLENTİLER
        engine.ts              (genel mekanik: özellik, check, zar atışı, sonuç)
        types.ts               (Ruleset arayüzü)
        rulesets/
          generic.ts           (basit/serbest varsayılan kural seti)
          dnd5e.ts             (D&D 5e eklentisi — kademeli)
          cyberpunk.ts         (Cyberpunk eklentisi — sonra)
      worlds/
        loader.ts              (JSON world definition okuma/doğrulama, ruleset bağlama)
        example-*.json
      domain/
        characterRepo.ts       (karakter oluşturma/kayıt/takip)
        campaignRepo.ts        (kampanya + oda + oyuncu üyeliği)
        sessionRepo.ts         (state, tur/aksiyon kuyruğu, event geçmişi)
      routes/
        worlds.ts, campaigns.ts, characters.ts, turns.ts
  clients/
    cli/                       (İLK HEDEF: terminal istemcisi — DM kalitesini olgunlaştırma)
    web/                       (sonra: tarayıcı arayüzü)
```

---

## 5. Veri Modeli (Postgres + pgvector)

Çok oyunculu + karakter + çoklu evren için genişletilmiş şema (kademeli kurulacak):

- `worlds (id, slug, title, definition JSONB, ruleset_id, created_at)`
- `campaigns (id, world_id, name, host_player_id, mode, status, created_at)`
  — `mode`: `'turn_based' | 'async'` (oda kurulurken seçilir)
- `players (id, handle, created_at)` — sisteme bağlanan kullanıcı/oyuncu
- `campaign_members (campaign_id, player_id, role)` — `role`: `'host' | 'player'`
- `characters (id, player_id, campaign_id, name, sheet JSONB, status, created_at)`
  — `sheet`: ruleset'e göre yapılandırılmış karakter sayfası (özellikler, HP, envanter)
- `sessions (id, campaign_id, state JSONB, current_turn, updated_at)`
  — paylaşılan dünya durumu (konumlar, NPC durumları, sahne)
- `actions (id, session_id, character_id, turn_no, raw_text, resolved, created_at)`
  — oyuncu aksiyon kuyruğu; turn_based & async ikisi de bunu kullanır
- `events (id, session_id, role, content TEXT, embedding vector(N), created_at)`
  — uzun vadeli hafıza/retrieval (embedding yerel Ollama'dan)
- `dm_decisions (id, session_id, action_id, dm_output JSONB, host_override JSONB, created_at)`
  — AI kararı + host'un düzeltmesi (gözetim izi)

> `schema.sql` içinde `CREATE EXTENSION IF NOT EXISTS vector;`. Embedding boyutu (`N`)
> seçilen yerel embedding modeline göre belirlenir (ör. `nomic-embed-text` = 768).

---

## 6. Oyun Döngüsü (Sıra Tabanlı / Asenkron Ortak Akış)

Her iki mod da aynı "aksiyon kuyruğu" üzerine kurulur; fark zamanlama ve bildirimdedir.

1. **Sahne kurulumu**: AI DM mevcut state + world + ruleset'ten bir sahne anlatır
   (herkese görünür anlatı).
2. **Aksiyon toplama**:
   - *Turn-based*: Tüm aktif oyuncular o tur için aksiyonunu bildirir (`actions` kuyruğu).
     Herkes bildirince (veya host tur kapatınca) tur işlenir.
   - *Async (play-by-post)*: Oyuncular istedikleri zaman aksiyon bırakır; DM uygun olunca
     biriken aksiyonları işler. Herkesin aynı anda online olması gerekmez.
3. **Çözümleme (resolution)**:
   - DM, `memory.ts` ile bağlam derler: world + güncel state + son olaylar +
     pgvector ile ilgili geçmiş olayların semantik retrieval'ı + gerekirse özet.
   - LLM `completeWithTools` ile çağrılır: hem anlatı üretir hem de yapılandırılmış
     tool-call'larla (`move`, `update_hp`, `give_item`, `roll_check`, `set_npc_state`...)
     state değişikliklerini bildirir.
   - `rules/engine.ts` + ilgili ruleset, check/zar gerektiren tool-call'ları hakemler.
   - `agent/oversight.ts`: host isterse kararı görür, onaylar / düzeltir / reddeder
     (`dm_decisions.host_override`).
4. **Uygulama & kayıt**: Doğrulanan tool-call'lar state'i günceller; olay `events`
   tablosuna (embedding ile) yazılır; anlatı tüm oyunculara döner.

---

## 7. Sağlayıcı-Bağımsız LLM Katmanı

Tek arayüz, çoklu implementasyon:

```ts
interface LLMProvider {
  complete(opts): Promise<string>;                    // serbest metin
  completeWithTools(opts): Promise<ToolUseResult>;    // tool-use döngüsü
  embed(texts: string[]): Promise<number[][]>;        // pgvector için
}
```

- **Varsayılan: Anthropic / Claude** (`claude-opus-4-8` veya uygun model) — ana DM mantığı,
  tool-use güvenilirliği ve tutarlılık/anlatı kalitesi için.
- **Alternatif: Ollama / Gemma** — config ile geçilir; geliştirme/test ve maliyetsiz
  denemeler için. **Uyarı:** küçük yerel modeller tool-use güvenilirliğinde zayıftır;
  bu yüzden kritik DM kararları için varsayılan Claude'dur.
- **Embedding: yerel Ollama** (`nomic-embed-text` vb.) — hafıza/retrieval için yeterli,
  bedava ve gizli.
- İleride **hibrit görev yönlendirme** (basit özet/NPC → yerel, kritik karar → Claude)
  bu katmanın üstüne eklenebilir; arayüz buna açık bırakıldı.

Konfigürasyon (`.env`): `LLM_PROVIDER=anthropic|ollama`, `ANTHROPIC_API_KEY`,
`OLLAMA_BASE_URL`, `EMBED_PROVIDER=ollama`, `EMBED_MODEL=nomic-embed-text`, `DATABASE_URL`.

---

## 8. Kural Motoru + Ruleset Eklentileri

```ts
interface Ruleset {
  id: string;                                  // "generic" | "dnd5e" | "cyberpunk"
  createCharacterSheet(input): CharacterSheet; // karakter oluşturma
  resolveCheck(sheet, check, context): CheckResult;  // yetenek kontrolü / zar
  applyConsequence(state, result): StatePatch; // sonucun state'e etkisi
  // ... savaş, hasar, ilerleme kademeli eklenir
}
```

- **`generic`** (ilk): hafif/serbest — basit özellik + d20-benzeri check. MVP buradan başlar.
- **`dnd5e`** (kademeli): gerçek karakter sayfası, atışlar, savaş.
- **`cyberpunk`** (sonra): kendi mekaniği.

DM, ruleset'i doğrudan çağırmaz; AI bir `roll_check` tool-call'u üretir, backend ilgili
ruleset ile **deterministik** olarak hakemlik eder (zar backend'de atılır → adil ve denetlenebilir).

---

## 9. MVP Kapsamı (İlk Hedef: AI DM Kalitesi, CLI üzerinden)

**Dahil:**
- Sağlayıcı-bağımsız LLM katmanı (`llm/`), varsayılan Claude, Ollama opsiyonu.
- `generic` ruleset + temel kural motoru.
- Yapılandırılmış state + tool-use + backend doğrulama.
- Hafıza: son N olay + özet + pgvector retrieval (yerel embedding).
- Karakter: basit oluşturma + kalıcı `sheet` (generic ruleset).
- Çok oyunculu temel: bir kampanya, birden çok karakter, **async** aksiyon kuyruğu.
- Host gözetimi: en azından kararı görme + override iskeleti.
- **CLI istemcisi**: tek makineden birden çok oyuncuyu simüle edip DM'i olgunlaştırma.

**Hariç (sonra):**
- Web ve Discord istemcileri.
- Gerçek zamanlı (WebSocket) eşzamanlı mod.
- D&D 5e / Cyberpunk tam rulesetleri (iskelet bırakılır).
- Hibrit LLM görev yönlendirme.

---

## 10. Doğrulama Yaklaşımı

- `docker compose up -d` ile Postgres+pgvector (+ ops. Ollama) ayağa kalkar; `schema.sql` uygulanır.
- CLI'dan iki "oyuncu" simüle edilerek bir `generic` dünyada async bir kampanya oynanır:
  sahne anlatımı → aksiyonlar → DM çözümlemesi → state güncellemesi (`psql` ile HP/envanter
  doğrulanır) → host override denemesi.
- `LLM_PROVIDER` Claude ↔ Ollama arasında değiştirilerek aynı akışın çalıştığı doğrulanır.
- Uzun oturum testinde pgvector retrieval'ın geçmiş bir olayı doğru hatırlattığı denenir.

---

## 11. Açık Sorular / Sonraki Görüşme Başlıkları

- Karakter oluşturma ne kadar rehberli olsun (AI sihirbazı mı, form mu)?
- Async modda "DM ne zaman işlesin" tetiği (host komutu mu, zamanlanmış mı)?
- Host override arayüzü CLI'da nasıl görünsün?
- Çoklu oyuncu kimlik/oturum yönetimi (basit handle mı, sonra gerçek auth mı)?

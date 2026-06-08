# Yazgi — AI Destekli Metin Macerası Motoru: İlk İskelet

## Context

**Hedef**: Klasik Zork tarzı metin maceralarını, statik senaryolar yerine AI ajanlarıyla dinamik olarak üreten, farklı tarz/hikayelerin "world definition" olarak eklenebildiği açık kaynak bir oyun motorunun temel iskeletini kurmak. Harita v1 kapsamında değil; ileride eklenebilecek şekilde mimari buna açık bırakılacak.

Karar verilmesi gereken tercihler: Node.js/TypeScript + Express backend, Claude API (Anthropic SDK), PostgreSQL + pgvector, ve **çift istemci modu** (basit terminal arayüzü + React/Vite SPA) — hangi istemcinin kullanılacağı oyuna/dünyaya göre seçilebilir.

## Mimari Genel Bakış

```
Yazgi/
  package.json                 (npm workspaces kökü)
  docker-compose.yml           (postgres + pgvector, lokal geliştirme)
  .env.example
  server/
    package.json, tsconfig.json
    src/
      server.ts                (Express app, route bağlama)
      db/
        client.ts              (pg Pool / client kurulumu)
        schema.sql             (worlds, sessions, events tabloları + pgvector extension)
      agent/
        gameMaster.ts          (Claude çağrısı: sistem promptu kurulumu + tool-use döngüsü)
        tools.ts               (yapılandırılmış state-mutation tool şemaları: move, take_item, vs.)
      worlds/
        loader.ts              (JSON world definition okuma/doğrulama)
        example-haunted-house.json  (örnek dünya — demo için)
      state/
        sessionRepo.ts         (Postgres'e state kaydet/yükle, event geçmişi)
      routes/
        worlds.ts              (GET /api/worlds)
        sessions.ts            (POST /api/sessions, GET /api/sessions/:id, POST /api/sessions/:id/command)
  clients/
    terminal/                  (build aracı yok — düz HTML/CSS/JS, fetch tabanlı)
      index.html, app.js, style.css
    react/                     (Vite + React SPA)
      src/App.tsx, ...
```

## Çekirdek Oyun Döngüsü Tasarımı

1. **World definition (JSON)**: dünya kimliği, başlık, ton/üslup talimatları (sistem promptu parçası), lokasyon grafiği (id, açıklama, çıkışlar, eşyalar, NPC'ler), başlangıç durumu, `preferredClient: "terminal" | "react"`.
2. **Session**: bir oyuncunun bir dünyadaki oturumu — Postgres'te `state JSONB` + olay geçmişi olarak saklanır.
3. **Komut akışı**:
   - Oyuncu serbest metin komutu gönderir (`POST /api/sessions/:id/command`)
   - Backend bağlam kurar: world definition + güncel state + son olaylar (uzun oturumlarda pgvector ile ilgili geçmiş olayları semantik arama)
   - Claude'a tool-use ile çağrı yapılır: model hem anlatı metni üretir hem de yapılandırılmış tool-call'larla (`move`, `take_item`, `talk_to_npc`, vb.) state değişikliklerini bildirir
   - Backend tool-call'ları world definition'a göre doğrular (örn. olmayan bir çıkışa hareket reddedilir), state'i günceller, olayı `events` tablosuna (embedding ile) ekler
   - Anlatı + güncellenmiş state istemciye döner

   Bu hibrit yaklaşım (serbest anlatı + yapılandırılmış tool-call doğrulaması) tutarlılığı korurken esnekliği sağlar — LLM'in tamamen serbest bırakılması state tutarsızlığına yol açar.

## Veritabanı Şeması (Postgres + pgvector)

- `worlds (id, slug, title, definition JSONB, preferred_client, created_at)`
- `sessions (id, world_id, state JSONB, created_at, updated_at)`
- `events (id, session_id, role, content TEXT, embedding vector(1536), created_at)` — uzun vadeli hafıza/retrieval için

`schema.sql` içinde `CREATE EXTENSION IF NOT EXISTS vector;` ile pgvector etkinleştirilecek.

## Çift İstemci Yaklaşımı

- İkisi de aynı REST API'yi tüketir, backend'de istemciye özel mantık yoktur.
- `clients/terminal`: build aracısız düz HTML/JS — klasik terminal/chat hissi, minimal bağımlılık.
- `clients/react`: Vite+React SPA — ileride envanter paneli, harita gibi zengin UI için genişlemeye uygun.
- Bir karşılama/seçim sayfası, dünyanın `preferred_client` alanına göre kullanıcıyı uygun istemciye yönlendirir; kullanıcı isterse manuel geçiş de yapabilir.

## API Uç Noktaları (v1)

- `GET /api/worlds` — mevcut dünyaları listele (id, başlık, preferred_client)
- `POST /api/sessions` — bir dünya için yeni oturum oluştur (başlangıç state ile)
- `GET /api/sessions/:id` — güncel state + son anlatıyı getir
- `POST /api/sessions/:id/command` — oyuncu komutu gönder, anlatı + güncellenmiş state al

## Uygulama Adımları

1. Kök `package.json` (npm workspaces: `server`, `clients/react`) + `.gitignore` güncellemesi (node_modules, dist, .env)
2. `docker-compose.yml` — `pgvector/pgvector:pg16` imajıyla Postgres servisi
3. `server/`: package.json, tsconfig, Express app iskeleti, `db/schema.sql` + `db/client.ts`
4. `worlds/loader.ts` + bir örnek world definition (`example-haunted-house.json`) — basit, birkaç odalı, tek NPC'li bir demo dünyası
5. `agent/tools.ts` — tool şemaları (move, take_item, drop_item, talk_to_npc, look) ve `agent/gameMaster.ts` — Anthropic SDK ile sistem promptu kurulumu, tool-use döngüsü, Claude yanıtının ayrıştırılması
6. `state/sessionRepo.ts` — Postgres CRUD (state kaydet/yükle, event ekle)
7. `routes/worlds.ts`, `routes/sessions.ts` ve `server.ts` route bağlama
8. `clients/terminal`: minimal HTML/CSS/JS — dünya seçimi, komut girişi, anlatı akışı
9. `clients/react`: Vite+React iskeleti — aynı akışı sağlayan temel bir SPA (zengin panel detayları ileride genişletilebilir)
10. `.env.example` (ANTHROPIC_API_KEY, DATABASE_URL) ve kök README güncellemesi (kurulum/çalıştırma talimatları)

## Doğrulama

- `docker compose up -d` ile Postgres+pgvector ayağa kaldırılır, `psql` ile `schema.sql` uygulanır (veya basit bir migration script'i ile)
- `cd server && npm run dev` ile backend başlatılır; `curl`/Postman ile `GET /api/worlds`, `POST /api/sessions`, `POST /api/sessions/:id/command` uçları manuel test edilir — örnek dünyada bir oda gezilip eşya alınarak state JSONB'nin doğru güncellendiği `psql` ile doğrulanır
- Terminal istemcisi tarayıcıda açılarak uçtan uca bir oyun turu (birkaç komut, lokasyon değişimi, eşya etkileşimi) denenir
- React istemcisi `npm run dev` (Vite) ile başlatılıp aynı temel akışın çalıştığı doğrulanır

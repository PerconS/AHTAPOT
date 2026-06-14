# AHTAPOT — Suç Hanedanlığı & Mafya Yaşam Simülasyonu
## Game Design + Software Architecture Document (v0.1)

> Bu doküman, kod yazımından önceki mimari sözleşmedir. Sonraki geliştirme
> adımlarında bu dokümandaki modüller, veri modelleri ve sınıf isimleri
> referans alınacaktır. Her bölüm, hem tasarım kararını hem de bunun
> yazılımsal karşılığını içerir.

---

## 0. Teknoloji Kararları (Tech Stack)

| Katman | Seçim | Gerekçe |
|---|---|---|
| Dil | **TypeScript (strict)** | Tip güvenliği, büyük domain modeli için zorunlu |
| Simülasyon Çekirdeği | **Saf TypeScript (headless, framework-bağımsız)** | Test edilebilir, deterministik, UI'dan tamamen ayrık |
| UI | **React 18 + Vite** | Bileşen tabanlı, hızlı HMR, geniş ekosistem |
| State (UI köprüsü) | **Zustand** | Hafif, boilerplate'siz, seçici subscription |
| Stil | **Tailwind CSS** | Hızlı prototip, tutarlı tasarım sistemi |
| Persistence | **IndexedDB (idb)** + JSON export | Büyük save dosyaları, tarayıcı kotası dostu |
| Rastgelelik | **Seeded PRNG (mulberry32/xoshiro)** | Deterministik simülasyon ve tekrar-üretilebilir save |
| Test | **Vitest** | Vite ile entegre, simülasyon birim testleri |
| Opsiyonel Backend | **Node + Fastify (faz 4)** | Bulut kayıt, leaderboard — çekirdek client-side kalır |

**Temel mimari prensip:** *Simülasyon çekirdeği UI'dan tamamen bağımsızdır.*
Çekirdek, Node üzerinde headless çalışabilir (hızlandırılmış test, denge analizi,
sunucu tarafı doğrulama). UI yalnızca bir görüntüleyici + komut göndericidir.

---

## 1. Core Gameplay Loop

### 1.1 Üst Düzey Döngü (Macro Loop)

```
        ┌──────────────────────────────────────────────┐
        │                                              │
        ▼                                              │
  [Planla] → [Yatırım/Eylem] → [Zaman İlerlet] → [Sonuç & Olay] → [Büyü/Adapte Ol]
        │                                                              │
        └──────────────── Hanedan Devri (Heir Succession) ◄───────────┘
```

1. **Planla:** Oyuncu bölge, suç operasyonu, personel ve ittifak kararları alır.
2. **Eylem/Yatırım:** Kaynak (para, etki, sadık adam) harcanır; operasyonlar kuyruğa alınır.
3. **Zaman İlerlet (Tick):** Dünya simülasyonu adımlar — gelir akar, ısı (heat) değişir, AI aileler hamle yapar.
4. **Sonuç & Olay:** Operasyon sonuçları çözülür, rastgele/koşullu olaylar tetiklenir.
5. **Büyü/Adapte Ol:** Yeni bölgeler, terfi eden adamlar, gelişen ekonomi.
6. **Hanedan Devri:** Patron öldüğünde/emekli olduğunda varis kontrolü devralır → **Legacy meta-loop**.

### 1.2 Çekirdek Tick Döngüsü (Micro Loop / Simulation Tick)

Sabit zaman adımı kullanılır. **1 tick = 1 oyun-içi gün** (ayarlanabilir).
Oyuncu zamanı duraklatabilir, hızlandırabilir (1x/2x/4x) veya "tek tick" adımlayabilir.

```
SimulationEngine.tick(dt):
  1. PRE_TICK        → zamanlanmış olayları (scheduler) topla
  2. PLAYER_COMMANDS → kuyruktaki oyuncu komutlarını uygula (Command pattern)
  3. ECONOMY_PHASE   → gelir/gider, vergi, kara para aklama
  4. CRIME_PHASE     → aktif operasyonların ilerlemesi & çözümü
  5. WORLD_PHASE     → polis ısısı, piyasa fiyatları, NPC hareketi
  6. AI_PHASE        → rakip ailelerin karar ağacı (kademeli, LOD'lu)
  7. TERRITORY_PHASE → bölge kontrolü, çatışma, sınır değişimi
  8. EVENT_PHASE     → koşul tabanlı + rastgele olay tetikleme
  9. RESOLUTION      → ölüm, tutuklama, terfi, ilişki güncellemeleri
 10. POST_TICK       → snapshot/dirty-flag, UI'a diff yayını
```

**Tasarım kararı:** Fazlar *deterministik sırada* çalışır ve hepsi tek bir
`GameState` üzerinde mutasyon yapan **System** sınıflarıdır (bkz. §2). Her faz
seeded PRNG'den türetilmiş kendi alt-stream'ini kullanır → tekrar-üretilebilirlik.

---

## 2. System Architecture

### 2.1 Katmanlı Mimari

```
┌─────────────────────────────────────────────────────────────┐
│ PRESENTATION LAYER (React + Zustand)                         │
│  Screens · Panels · HUD · Map · Dialogs                      │
│  ──────────── selectors ▲ | ▼ dispatch(Command) ──────────  │
├─────────────────────────────────────────────────────────────┤
│ APPLICATION LAYER (Game Facade / Command Bus)               │
│  GameController · CommandQueue · EventBus · TimeController   │
├─────────────────────────────────────────────────────────────┤
│ SIMULATION CORE (saf TypeScript, headless, deterministik)   │
│  SimulationEngine (tick loop)                               │
│  Systems:  EconomySystem · CrimeSystem · WorldSystem        │
│            AISystem · TerritorySystem · EventSystem         │
│            DynastySystem · RelationshipSystem               │
│  GameState (single source of truth)                         │
│  Entities/Components · RNG · Scheduler · RuleConfig          │
├─────────────────────────────────────────────────────────────┤
│ PERSISTENCE LAYER                                            │
│  SaveManager · Serializer · MigrationRunner · StorageAdapter│
│  (IndexedDB | File export | Cloud adapter)                  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Veri Akışı (tek yönlü)

```
UI Event → Command → CommandQueue → SimulationEngine.tick()
   → GameState mutation → EventBus emit → Zustand store update → UI re-render
```

- **Komutlar (Command pattern):** Tüm oyuncu eylemleri serileştirilebilir
  komut nesneleridir (`{ type, payload, issuedAtTick }`). Bu, *replay*,
  *undo* (sınırlı), ve *deterministik test* sağlar.
- **EventBus:** Sistemler birbirini doğrudan çağırmaz; olay yayınlar
  (`crime.completed`, `boss.died`, `territory.lost`). Gevşek bağlılık.
- **Selectors:** UI, `GameState`'in tamamına değil; memoize edilmiş
  seçicilere abone olur (re-render minimizasyonu).

### 2.3 Modül Sınırları (Dizin Yapısı)

```
/src
  /core
    /engine        SimulationEngine, Scheduler, RNG, Clock
    /state         GameState, store factory, snapshot
    /entities      Family, Character, Territory, Operation, Business
    /systems       *System.ts (her biri tek sorumluluk)
    /rules         RuleConfig (denge sabitleri, data-driven)
    /events        EventBus, EventDefinitions, EventTriggers
  /app
    GameController, CommandBus, command-handlers, TimeController
  /persistence
    SaveManager, Serializer, migrations/, StorageAdapter
  /ui
    /screens /components /hud /map /dialogs /hooks /store(zustand)
  /data            JSON: bölgeler, suç tipleri, isimler, olay tablo
  /shared          types, math, id, result<T,E>
  /test            simülasyon senaryoları, denge testleri
```

---

## 3. Class Diagram

```
                          ┌────────────────────┐
                          │  SimulationEngine   │
                          │  + tick(dt)         │
                          │  - systems[]        │
                          │  - rng: RNG         │
                          │  - clock: Clock     │
                          └─────────┬──────────┘
                                    │ orchestrates
            ┌───────────┬───────────┼───────────┬───────────┐
            ▼           ▼           ▼           ▼           ▼
      ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐ ┌──────────┐
      │Economy  │ │  Crime   │ │ World   │ │  AI    │ │Territory │  (Systems)
      │System   │ │ System   │ │ System  │ │ System │ │ System   │
      └────┬────┘ └────┬─────┘ └────┬────┘ └───┬────┘ └────┬─────┘
           └───────────┴────── read/write ─────┴───────────┘
                                    │
                                    ▼
                          ┌────────────────────┐
                          │     GameState       │  (single source of truth)
                          │  - families: Map    │
                          │  - territories: Map │
                          │  - operations: Map  │
                          │  - market: Market   │
                          │  - world: WorldState│
                          │  - clock, rngState  │
                          └─────────┬──────────┘
                                    │ contains
      ┌──────────────┬──────────────┼──────────────┬──────────────┐
      ▼              ▼              ▼              ▼              ▼
┌──────────┐  ┌────────────┐  ┌───────────┐  ┌──────────┐  ┌──────────┐
│  Family  │  │ Character  │  │ Territory │  │Operation │  │ Business │
├──────────┤  ├────────────┤  ├───────────┤  ├──────────┤  ├──────────┤
│+id       │  │+id         │  │+id        │  │+id       │  │+id       │
│+name     │  │+role       │  │+ownerId   │  │+type     │  │+type     │
│+treasury │  │+stats      │  │+income    │  │+crewIds  │  │+income   │
│+heat     │  │+loyalty    │  │+heat      │  │+risk     │  │+launder  │
│+memberIds│  │+traits[]   │  │+adjacency │  │+progress │  │+cover    │
│+isPlayer │  │+familyId   │  │+upgrades  │  │+status   │  │+heat     │
│+aiProfile│  │+age,health │  │+defense   │  │+reward   │  │          │
│+relations│  │+relations  │  └───────────┘  └──────────┘  └──────────┘
└────┬─────┘  └────────────┘
     │ has-a (player only)
     ▼
┌──────────────┐         ┌──────────────────┐
│ Dynasty      │         │ AIProfile        │  (rakip aile beyni)
├──────────────┤         ├──────────────────┤
│+bossId       │         │+archetype        │
│+heirs[]      │         │+aggression       │
│+legacyPoints │         │+greed,caution    │
│+perks[]      │         │+goals[]          │
│+generation   │         │+behaviorTree     │
│+history[]    │         │+memory(grudges)  │
└──────────────┘         └──────────────────┘
```

**Sınıf sorumlulukları (özet):**

- **SimulationEngine** — tick orkestrasyonu; sistemleri sırayla çalıştırır. Durum tutmaz dışında RNG/clock.
- **System (abstract)** — `update(state, ctx)` arayüzü. Her sistem saf fonksiyon gibi davranır: state'i alır, mutasyon yapar/olay yayar.
- **GameState** — tüm veri; serileştirilebilir, davranış içermez (anemic by design).
- **Entity sınıfları** — veri kapsayıcı (POJO/struct), ağır davranış sistemlerde.
- **GameController** — UI ile çekirdek arasında facade; komut dispatch + zaman kontrolü.

---

## 4. Data Models

> Tüm modeller **serileştirilebilir** (fonksiyon/Map yerine düz veri; runtime'da
> `Map` kullanılır, persist sırasında array'e çevrilir). `Id` = branded string.

```ts
type Id = string & { readonly __brand: unique symbol };
type Tick = number;       // mutlak oyun günü
type Money = number;      // tam sayı (kuruş cinsinden tutmak overflow'u önler)

interface GameState {
  meta: { version: number; seed: number; createdAt: number; saveName: string };
  clock: { tick: Tick; speed: 0|1|2|4; paused: boolean };
  rngState: RngState;                 // PRNG durumu (deterministik resume)
  families: Record<Id, Family>;
  characters: Record<Id, Character>;
  territories: Record<Id, Territory>;
  operations: Record<Id, Operation>;
  businesses: Record<Id, Business>;
  market: Market;
  world: WorldState;
  dynasty: Dynasty;                   // oyuncunun hanedanı
  eventLog: GameEvent[];              // dairesel tampon (son N olay)
  flags: Record<string, number>;     // story/quest bayrakları
}

interface Family {
  id: Id;
  name: string;
  isPlayer: boolean;
  treasury: Money;
  reputation: number;        // 0..100 sokak itibarı
  heat: number;              // 0..100 polis baskısı
  memberIds: Id[];
  territoryIds: Id[];
  businessIds: Id[];
  relations: Record<Id, Relation>;   // diğer ailelerle ilişki
  aiProfile?: AIProfile;             // sadece AI ailelerde
}

interface Character {
  id: Id;
  familyId: Id | null;
  name: string;
  role: Role;                 // BOSS|UNDERBOSS|CAPO|SOLDIER|ASSOCIATE
  stats: { combat: number; cunning: number; charisma: number; loyalty: number; management: number };
  traits: Trait[];            // 'ruthless','hot-headed','loyal','greedy'...
  age: number; health: number;
  status: 'active'|'jailed'|'hospital'|'dead'|'hiding';
  isHeir: boolean;
  relations: Record<Id, number>;     // kişisel bağlar (-100..100)
}

interface Territory {
  id: Id;
  name: string;
  ownerId: Id | null;
  baseIncome: Money;
  heat: number;               // bölgesel polis ilgisi
  defense: number;            // savunma puanı
  adjacency: Id[];            // komşu bölgeler (graf)
  upgrades: Upgrade[];
  contested: boolean;         // aktif çatışma var mı
}

interface Operation {            // aktif suç işi
  id: Id; familyId: Id;
  type: CrimeType;
  crewIds: Id[];              // atanan karakterler
  targetId?: Id;             // bölge/aile/iş
  risk: number;              // 0..1
  progress: number;          // 0..1
  durationTicks: number;
  status: 'planned'|'active'|'success'|'failed'|'busted';
  expectedReward: Money;
}

interface Business {             // meşru/yarı-meşru cephe
  id: Id; ownerId: Id;
  type: BusinessType;        // restaurant, casino, laundromat, club...
  income: Money;
  launderCapacity: Money;    // tick başına aklanabilir kara para
  coverStrength: number;     // ısıyı maskeleme
  heat: number;
}

interface Market {            // ekonomi durumu
  goods: Record<GoodType, { price: Money; supply: number; demand: number; volatility: number }>;
  trends: { policePressure: number; economy: number };
}

interface WorldState {
  policeHeatGlobal: number;
  newsFeed: NewsItem[];
  factions: Record<Id, { influence: number }>;  // polis, yargı, siyaset
  date: { year: number; day: number };
}

interface Dynasty {
  bossId: Id;
  heirs: Id[];
  generation: number;
  legacyPoints: number;
  perks: LegacyPerk[];
  history: DynastyChapter[];   // anlatısal kayıt
}

interface AIProfile {
  archetype: 'aggressor'|'merchant'|'schemer'|'turtle'|'opportunist';
  aggression: number; greed: number; caution: number; vengeance: number;
  goals: AIGoal[];
  grudges: Record<Id, number>;  // hafıza: kime ne kadar kin
}

interface Relation { standing: number; treaty?: 'peace'|'alliance'|'war'|'tribute'; }
```

**Tasarım notları:**
- `Record<Id, T>` runtime'da `Map`'e mirror'lanır (O(1) erişim); persist'te düz objeye iner.
- Modeller **anemic** (davranışsız) — tüm kurallar System'lerde, böylece save dosyası saf veridir ve versiyon migrasyonu kolaydır.
- `rngState` kayıtla beraber tutulur → yüklendiğinde simülasyon birebir aynı devam eder.

---

## 5. World Simulation Design

### 5.1 Yaşayan Dünya Prensibi
Dünya, oyuncu olmadan da ilerler. Her tick'te `WorldSystem` şunları işler:

- **Polis Isısı (Heat):** Global + bölgesel. Suç → ısı artar; zaman + rüşvet → düşer. Eşik aşılırsa **baskın/operasyon** olayları tetiklenir.
- **Fraksiyonlar:** Polis, yargı, siyaset, basın. Her birinin "influence" değeri; oyuncu rüşvet/şantajla manipüle edebilir.
- **Haber Akışı (NewsFeed):** Simülasyon olaylarından üretilen anlatısal metinler — dünyaya hayat hissi katar ve oyuncuya istihbarat verir.
- **Takvim & Mevsimsellik:** Bazı suçlar/işler dönemsel (tatil sezonu kaçakçılık ↑, vb.).

### 5.2 Level of Detail (LOD) — Ölçeklenebilirlik
Dünyada onlarca aile ve yüzlerce karakter olabilir. Maliyeti sınırlamak için:

| LOD | Kapsam | İşlem Sıklığı |
|---|---|---|
| **Full** | Oyuncu + komşu/rakip aktif aileler | Her tick |
| **Coarse** | Uzaktaki aileler | N tick'te bir, basitleştirilmiş model |
| **Statistical** | Arka plan dünya | Toplu istatistiksel güncelleme (Monte Carlo lite) |

Yakınlık (bölge grafı mesafesi) ve oyuncuyla ilişki LOD'u belirler.

---

## 6. Economy Design

### 6.1 Para Akış Modeli (Kapalı Devre)
```
GELİR                         GİDER
─────                         ─────
Bölge haraçları           →   Personel maaşı (sadakat ↔ ödeme)
Suç operasyon kârı        →   Bölge bakım/upgrade
Meşru iş geliri           →   Rüşvet (ısı azaltma)
Kara para aklama çıktısı   →   Silah/teçhizat
Faiz/tefecilik            →   Savaş maliyeti (çatışma)
                              Avukat/kefalet (tutuklamada)
```

### 6.2 İki Para Türü: Temiz vs Kirli
- **Dirty Money:** Suçtan gelir; doğrudan harcanırsa ısı yaratır, sınırlı kullanım.
- **Clean Money:** Aklanmış; serbestçe yatırım/genişleme için kullanılır.
- **Aklama (Laundering):** `Business.launderCapacity` tick başına dönüşüm; kapasiteyi aşan kirli para birikir ve risk yaratır. Aklama oranında "kesinti" (örn. %15 kayıp).

### 6.3 Dinamik Piyasa
- Mal fiyatları arz/talep + volatilite ile dalgalanır (mean-reverting random walk).
- Oyuncu eylemleri piyasayı etkiler (bir bölgeyi domine etmek fiyat ↑).
- **Denge sabitleri data-driven** (`/rules`), kod değişmeden tuning yapılır.

### 6.4 Ekonomik Geri Besleme Döngüleri
- Pozitif: Bölge → gelir → güç → daha çok bölge (kontrolsüz büyümeyi **ısı** ve **AI baskısı** dengeler).
- Negatif: Yüksek ısı → rüşvet maliyeti ↑ → kâr ↓; aşırı genişleme → savunma seyrelir.

---

## 7. Territory System Design

### 7.1 Graf Tabanlı Harita
Bölgeler bir **graf** olarak modellenir (`adjacency`). Genişleme yalnızca
komşu bölgelere yapılabilir → coğrafi strateji.

### 7.2 Bölge Kontrol Mekanikleri
- **Ele Geçirme:** Operasyon (sızma, sindirme, açık savaş) ile. Başarı =
  saldıran güç vs `defense` + savunucu takviye.
- **Kontrol Derecesi:** Bölge tam/kısmi kontrol edilebilir (`contested` durumu).
- **Upgrade'ler:** Savunma, gelir, ısı-azaltma, aklama tesisi.
- **Sınır Çatışması:** Komşu rakip bölgeler sürekli düşük yoğunluklu baskı yaratır.

### 7.3 Çatışma Çözümü (Territory Phase)
```
resolveConflict(attacker, defender):
  attackPower  = Σ crew(combat·morale) · weapons · surpriseBonus
  defendPower  = territory.defense + Σ garrison(combat) · homeBonus
  outcome      = stochastic(attackPower, defendPower, rng)
  → kayıplar (karakter ölüm/yaralanma), ısı artışı, kontrol değişimi
```
Sonuç olasılıksaldır ama güç farkına göre ağırlıklı → "underdog" sürprizleri mümkün.

---

## 8. Crime System Design

### 8.1 Suç Operasyonu Yaşam Döngüsü
```
PLAN → ekip ata + hedef seç → RİSK hesapla → ACTIVE (progress/tick)
   → RESOLVE: success | failure | busted → ÖDÜL/CEZA + ısı + ilişki etkisi
```

### 8.2 Suç Tipleri (data-driven katalog)
| Tip | Risk | Ödül | Isı | Gereken Yetenek |
|---|---|---|---|---|
| Haraç (extortion) | düşük | düşük-orta | düşük | charisma/combat |
| Soygun (heist) | yüksek | yüksek | yüksek | cunning/combat |
| Kaçakçılık (smuggling) | orta | orta-yüksek | orta | cunning/management |
| Tefecilik (loan shark) | düşük | sürekli | düşük | charisma |
| Suikast (hit) | yüksek | stratejik | yüksek | combat/cunning |
| Rüşvet/şantaj | orta | etki | negatif ısı | cunning/charisma |

### 8.3 Risk & Sonuç Modeli
```
risk = base(crimeType)
     + heatFactor(family.heat, territory.heat)
     - crewSkill(relevantStats)
     - prepBonus(zaman/para yatırımı)
successChance = clamp(1 - risk, 0.05, 0.95)
roll = rng.next()
  roll < successChance              → SUCCESS
  successChance ≤ roll < success+ε   → FAILURE (ödül yok, az ısı)
  else                               → BUSTED (tutuklama + yüksek ısı)
```

### 8.4 Sonuçların Yayılımı
Her operasyon `EventBus`'a yayar: ısı sistemi, ekonomi, ilişki ve AI hafızası
(grudge) bu olaylara reaksiyon verir. Bir suikast → hedef ailenin `vengeance` ↑.

---

## 9. Dynasty System Design

### 9.1 Meta-Loop: Nesiller
Patron **ölür** (yaş/health/suikast) veya **emekli** olur → **varis** kontrolü
devralır. Oyun biter değil; **devam eder** → uzun vadeli "dynasty" oyunu.

### 9.2 Varis (Heir) Sistemi
- Aile üyeleri arasından varis(ler) yetiştirilir (eğitim → stat artışı).
- Varis seçimi: stat, sadakat, kan bağı, oyuncu tercihi.
- Boş varis = **Game Over** (hanedan çöker) → yeni oyun, ama **Legacy** kalır.

### 9.3 Legacy Points & Perks (Roguelite Meta-Progression)
```
Nesil sonunda kazanılan LegacyPoints =
   f(toplam servet, bölge sayısı, hayatta kalınan yıl, başarılan kilometre taşları)
→ kalıcı perk ağacında harcanır:
   • Başlangıç sermayesi ↑   • İlk sadık adamlar
   • Isı kazanımı ↓          • Aklama verimi ↑
   • Varis stat bonusu        • Diplomasi açılışları
```
Perks **hanedanlar arası** taşınır → her yeni başlangıç biraz daha güçlü.

### 9.4 Dynasty History
`DynastyChapter[]` — her nesil için anlatısal özet (patronun adı, hüküm yılları,
zirve serveti, ölüm şekli). Oyuncuya "tarih kitabı" hissi verir; emergent storytelling.

---

## 10. UI Architecture

### 10.1 Ekran Yapısı
```
App
 ├─ MainMenu (New / Load / Legacy Tree / Settings)
 ├─ GameShell
 │   ├─ TopBar      (para temiz/kirli, ısı, tarih, hız kontrolü)
 │   ├─ MapView     (bölge grafı — kontrol/ısı renk kodlu)
 │   ├─ SidePanel   (sekmeler: Aile · Operasyonlar · İşler · Diplomasi)
 │   ├─ EventModal  (olay/karar diyalogları)
 │   └─ NewsTicker  (dünya haber akışı)
 └─ GameOver / SuccessionScreen
```

### 10.2 State Köprüsü (Zustand)
- Çekirdek `GameState` **tek kaynaktır**; UI store onun **read-model**'idir.
- `SimulationEngine` her tick sonunda **diff/snapshot** yayınlar → Zustand günceller.
- UI asla `GameState`'i doğrudan mutasyona uğratmaz; **yalnızca Command dispatch eder**.
- **Selector + memoization** ile yalnızca değişen panel re-render olur.

### 10.3 Komut Akışı (UI → Core)
```
<button onClick={() => dispatch(StartOperation({crimeType, crewIds, targetId}))}>
   → CommandBus → CommandQueue → bir sonraki tick'te işlenir
```
Bu, UI'ı simülasyon hızından ayırır ve replay/test'i mümkün kılar.

---

## 11. Save/Load Architecture

### 11.1 Prensipler
- `GameState` tasarım gereği **tam serileştirilebilir** (davranış yok, sadece veri).
- `rngState` + `clock` kaydedildiği için yükleme **birebir deterministik** devam eder.
- Save = `{ version, checksum, compressedState }`.

### 11.2 Pipeline
```
SAVE:  GameState → Serializer(Map→array) → JSON → (opsiyonel) LZ-compress
       → checksum ekle → StorageAdapter.write(slot)
LOAD:  StorageAdapter.read → checksum doğrula → decompress → parse
       → MigrationRunner(oldVersion→current) → hydrate(array→Map) → GameState
```

### 11.3 Versioning & Migration
```
migrations/
  001_initial.ts
  002_add_dynasty_perks.ts
  003_split_money_clean_dirty.ts
MigrationRunner: state.meta.version'dan CURRENT_VERSION'a sıralı uygular.
```
Her şema değişikliği bir migration ekler → eski save'ler bozulmaz.

### 11.4 Save Slotları & Adapter
- **StorageAdapter** arayüzü: `IndexedDBAdapter` (varsayılan), `FileAdapter`
  (JSON export/import), `CloudAdapter` (faz 4).
- Çoklu slot, autosave (her N tick), manual save, "iron man" (tek slot) modu.

---

## 12. Performance Strategy

| Alan | Strateji |
|---|---|
| **Simülasyon** | Saf TS, allocation minimizasyonu; sistemler dirty-set üzerinde çalışır (her tick her entity değil) |
| **AI maliyeti** | LOD (§5.2) — uzak aileler seyrek/istatistiksel işlenir |
| **Veri erişimi** | Runtime'da `Map` (O(1)); graf komşulukları önceden hesaplı |
| **UI render** | Zustand seçici subscription + `React.memo` + virtualized listeler |
| **Tick/UI ayrımı** | Simülasyon UI thread'ini bloklarsa → **Web Worker**'a taşınabilir (çekirdek headless olduğu için maliyetsiz) |
| **Save boyutu** | Map→array sıkıştırma, eventLog dairesel tampon (sınırlı), LZ compress |
| **Determinizm** | Seeded PRNG → ağır denge testleri headless & hızlandırılmış çalışır |
| **Profiling** | Faz bazlı tick süresi ölçümü (her System'in maliyeti loglanır) |

**Kritik karar:** Çekirdeğin headless ve framework-bağımsız olması, ileride
simülasyonu bir **Web Worker**'a veya **Node sunucuya** taşımayı sıfır refactor'la
mümkün kılar — bu en büyük ölçeklenebilirlik kaldıracıdır.

---

## 13. Development Roadmap

### Faz 0 — İskelet (Foundation)
- Repo, TS strict, Vite, Vitest, lint/format kurulumu.
- `GameState`, temel tipler, `RNG`, `Clock`, boş `SimulationEngine.tick()`.
- **Çıktı:** Headless boş tick çalışır, deterministik seed testi geçer.

### Faz 1 — Çekirdek Simülasyon
- `EconomySystem` (gelir/gider), `CrimeSystem` (operasyon yaşam döngüsü).
- `CommandBus` + temel komutlar. Minimal CLI/test harness.
- **Çıktı:** Headless'ta para kazanılıp harcanabilir; bir suç tam döngü çözülür.

### Faz 2 — Dünya & Bölge
- `TerritorySystem` (graf, ele geçirme, çatışma), `WorldSystem` (ısı, fraksiyon).
- `EventSystem` (koşullu + rastgele olaylar).
- **Çıktı:** Yaşayan bir dünyada bölge genişlemesi ve ısı yönetimi.

### Faz 3 — UI Entegrasyonu
- React shell, MapView, pan’ller, Zustand köprüsü, zaman kontrolü.
- `SaveManager` + IndexedDB + migration iskeleti.
- **Çıktı:** Oynanabilir tek-nesil dikey dilim (vertical slice).

### Faz 4 — AI Rakip Aileler
- `AISystem` (archetype + behavior tree + grudge hafızası), LOD.
- Diplomasi (treaty/war/alliance) ekranı.
- **Çıktı:** Rakipler proaktif hamle yapar, oyuncuya tepki verir.

### Faz 5 — Hanedan & Legacy
- `DynastySystem` (succession, heir, legacy points/perks, history).
- Meta-progression ekranları, game-over/succession akışı.
- **Çıktı:** Tam meta-loop; nesiller arası oyun.

### Faz 6 — Cila & Denge
- Data-driven tuning, ekonomi dengesi, içerik genişletme, ses/animasyon.
- Headless denge simülasyonları (1000x hızlandırılmış oto-oyun testleri).
- Opsiyonel: bulut kayıt, Web Worker'a simülasyon taşıma.

---

## Ek: Yerleşik Mimari Kararlar Özeti (ADR-lite)

1. **Çekirdek UI'dan ayrık & headless** → test, performans, taşınabilirlik.
2. **Anemic data model + System'ler** → save/migration kolaylığı, ECS-benzeri ölçek.
3. **Command pattern + tek yönlü akış** → replay, determinism, undo.
4. **Seeded PRNG + rngState persist** → birebir tekrar-üretilebilir simülasyon.
5. **EventBus ile gevşek bağlılık** → sistemler bağımsız geliştirilir/test edilir.
6. **Data-driven kurallar (`/rules`, `/data`)** → kod değişmeden denge tuning.
7. **LOD'lu dünya simülasyonu** → onlarca aile, yüzlerce NPC ölçeklenir.

---

*Sonraki adım: Bu mimari onaylandığında Faz 0 iskeletini kurabiliriz.
Onayını bekliyorum — başlamamı istediğin fazı belirt.*

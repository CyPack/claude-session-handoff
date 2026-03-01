---
name: session-handoff
version: 2.5.0
maturity: opus
maturity_score: 8
lesson_count: 0
last_updated: 2026-03-01
description: Session arasi bilgi kaybini sifira indiren gecis promptu protokolu
triggers:
  - handoff
  - handoff yap
  - gecis
  - gecis promptu
  - gecis hazirla
  - session bitiyor
  - devam ederiz
  - sonraki chat
  - sonraki session
  - bunu kaydet
  - session'a aktar
  - sonraki chat icin prompt hazirla
  - sonraki chat'te devam
---

# Session Handoff Protocol (GLOBAL - ZORUNLU)

> Bu protokol OTONOM calisir. Kullanici soylemese bile context sikisma sinyalleri
> tespit edildiginde Claude KENDI BASINA handoff promptu uretmeye baslar.
> Sonraki session'daki Claude'u TETIKLEYECEK kadar detayli bir gecis promptu uretir.

---

## CREDENTIAL BLOCK

| Alan | Deger |
|------|-------|
| **Olusturan** | Ayaz + Claude Code (Opus), iteratif co-creation |
| **Gecmis** | 3 session, 5 iterasyon (template → rule → otonom → pipeline → audit) |
| **Kullanan** | Her Claude Code session'indaki Claude instance |
| **Hedef** | Sonraki session'daki sifir-context Claude |
| **Amac** | Session arasi bilgi kaybini sifira indirmek |
| **Scope** | Global (`~/.claude/rules/`), tum projeler, tum skill'ler |
| **Limitler** | Claude model bagimli; lesson kalite filtresi confidence-based; test framework yok |
| **Dogrulama** | Cross-check audit v2.1 (3 agent, 9 rules, 15 fix) |

### Version History

| Versiyon | Tarih | Degisiklik | Dogrulama |
|----------|-------|------------|-----------|
| v2.0.0 | 2026-02-27 | Ilk surum: otonom tetikleme + 6 HAZIRLIK + 12 bolum prompt + 44+ trigger | Self-test |
| v2.0.1 | 2026-02-27 | Katman 1: HAZIRLIK rename, aktif skill tespiti, cift referans fix (3 fix) | grep 3/3 PASS |
| v2.1.0 | 2026-02-27 | Katman 2-4: Cross-check audit 15 fix, bolme karari (A), 9 rules uyumu | 14/14 grep PASS |
| v2.2.0 | 2026-02-28 | Credential block + lesson kalite kontrol (confidence tag + celiski tespiti) | — |
| v2.3.0 | 2026-02-28 | BOLUM 2: manifest-based akilli context yukleme (3 katman: auto-load/zorunlu/on-demand) | grep 4/4 PASS |
| v2.4.0 | 2026-02-28 | Otonom tetikleme KALDIRILDI → manuel. Per-handoff metadata (BOLUM 1 + /tmp/ frontmatter). HAZIRLIK 2'ye trigger_reason eklendi. | — |
| v2.5.0 | 2026-02-28 | Task State Serialization (BOLUM 5.5), TaskList Generator + Specialist Agent Generator (BOLUM 6), HAZIRLIK 2+5 task snapshot | grep 5/5 PASS |

---

## MANUEL TETIKLEME MEKANIZMASI (SADECE KULLANICI ISTEDIGINDE)

### Iron Law: Handoff SADECE Kullanici Istediginde

```
OTONOM HANDOFF YASAK — context dolulugu TEK BASINA tetikleyici DEGILDIR.
Handoff SADECE kullanici asagidaki ifadelerden birini kullandiginda baslar:
1. "handoff yap" / "gecis promptu olustur" / "gecis hazirla"
2. "sonraki chat icin prompt hazirla" / "sonraki session'a aktar"
3. "session bitiyor, kaydet" / "bunu kaydet"
4. "devam ederiz" / "sonraki chat'te devam"

KULLANICI ISTEMEDIKCE:
- Handoff hazirligina BASLA
- Token'lari handoff icin HARCAMA
- Cevaplari kisaltma bahanesiyle handoff'a YONLENDIRME

HANDOFF BASLADIGINDA:
- TOKEN CIMRILIGI ASLA YAPMA — bu prompt sonraki session'in TEK BAGLAMI
- KISALTMA, OZET, SUMMARY YASAK — her detay, her dosya yolu, her sayi yazilmali
```

### Kullanici Istediginde Yapilacaklar (6 HAZIRLIK — SIRASYLA)

```
HAZIRLIK 1: Kullaniciya bildir:
  "Gecis hazirligina basliyorum.
   Once dosyalari guncelleyecegim, sonra gecis promptunu hazirlayacagim.
   Token tasarrufu yapmiyorum — sonraki session'in tek baglami bu olacak."
```

```
HAZIRLIK 2: Session state'i topla (hafizada tut, henuz yazma):
  - Trigger reason tespit et:
    | Deger | Kosul |
    | explicit_user | Kullanici acikca "handoff yap" dedi |
    | milestone_complete | Buyuk gorev/phase tamamlandi, dogal gecis |
    | mid_task | Gorev ortasinda, devam edilecek is var |
  - Ne yapildi? (tamamlanan gorevler, commitler, testler)
  - Ne kaldi? (acik gorevler, bloklanan isler)
  - Ne ogrenildi? (yeni hatalar, kesfedilen patternler)
  - Ne karar verildi? (mimari kararlar, trade-off'lar)
  - Ne ertelendi? (bilerek yapilmayan seyler)
  - Hangi dosyalar olusturuldu/degistirildi?
  - Hangi servisler test edildi, hangi sonuclar alindi?
  - Aktif skill'leri tespit et (session'da kullanilan skill'ler, [] ise bos)
  - Mevcut TaskList snapshot'i al (TaskList cagir, in_progress + pending task'lari kaydet)
  MCP VERI NOTU: MCP tool'dan gelen veriler (task listesi, API sonuclari vb.)
  prompt'a yazilirken [kaynak: MCP tool, session-icinde] olarak etiketlenmeli.
  Hafizadan MCP verisi uretme — mcp-anti-hallucination.md gecerli.
```

```
HAZIRLIK 3: DOSYA GUNCELLEME PIPELINE (PROMPT URETMEDEN ONCE — ZORUNLU)

Asagidaki dosyalari SIRASYLA guncelle. Her guncelleme Edit ile yapilir, ASLA Write degil.
Dosya yoksa ATLA (olusturma — bu handoff'un isi degil).
Session'da yeni bulgu YOKSA o dosyayi ATLA (gereksiz guncelleme yapma).

AKTIF SKILL TESPITI:
Asagidaki kosullardan herhangi biri gecerliyse o skill "aktif"tir:
- Session sirasinda Skill tool ile o skill cagirildi
- ~/.claude/skills/{skill}/lessons/ dosyalari okundu
- Kullanici o skill'in konusuyla ilgili calisma yapti (ornek: bridge → openclaw-manage)
- Session basinda handoff promptunda o skill referans verildi
Hicbir skill aktif degilse SIRA 1-3 ATLA, sadece SIRA 4-5 uygula.

SIRA 1 — Lesson Dosyalari (aktif skill varsa):
┌─────────────────────────────────────────────────────────────────────────┐
│ Dosya                              │ Ne Guncellenir                    │
├─────────────────────────────────────┼───────────────────────────────────┤
│ ~/.claude/skills/{skill}/           │ Session'da bulunan YENI hatalar   │
│   lessons/errors.md                 │ Format: | hata | neden | fix |   │
│                                     │ tarih | (tablo satiri, prose YOK) │
├─────────────────────────────────────┼───────────────────────────────────┤
│ ~/.claude/skills/{skill}/           │ Session'da kesfedilen YENI        │
│   lessons/golden-paths.md           │ basarili workflow'lar             │
│                                     │ Format: | senaryo | adimlar |    │
│                                     │ on-kosul | tarih |               │
├─────────────────────────────────────┼───────────────────────────────────┤
│ ~/.claude/skills/{skill}/           │ Session'da yasanan YENI           │
│   lessons/edge-cases.md             │ edge case'ler                    │
│                                     │ Format: | durum | cozum | tarih │
├─────────────────────────────────────┼───────────────────────────────────┤
│ ~/.claude/skills/_shared/           │ Cross-skill pattern varsa         │
│   common-errors.md                  │ (MCP, auth, docker, config,      │
│                                     │  playwright/browser)             │
└─────────────────────────────────────┴───────────────────────────────────┘

DUPLIKAT ONLEME: Phase 3 Post-Execution bu session'da ZATEN lesson yazdiysa,
ayni kaydi tekrar yazma. "Yeni bulgu" = Phase 3 tarafindan henuz YAZILMAMIS bulgu.
Phase 3 zaten yazdigi kayitlar duplikat kontrolunde EXCLUDE edilir.

LESSON KALITE KONTROL:
Her lesson yazilirken confidence seviyesi ZORUNLU olarak eklenir:

| Seviye | Kriter | Format | TTL |
|--------|--------|--------|-----|
| HIGH | Kullanici duzeltmesi, 2+ session'da tekrarlanan, test ile dogrulanan | `[H]` | Sinirsiz |
| MEDIUM | Tek session'da gozlemlenen, mantikli ama tekrarlanmamis | `[M]` | Sinirsiz |
| LOW | Spekulatif, test edilmemis, tek veri noktasi | `[L]` | 3 session |

Kurallar:
- Her lesson satirinin SONUNA `[H]`, `[M]` veya `[L]` etiketi ekle
  Ornek: `| auth timeout | token expire | refresh token | 2026-02-28 | [M] |`
- Kullanici duzeltmesi = otomatik `[H]` (kullanici her zaman hakli)
- `[L]` etiketli lesson 3 session boyunca DOGRULANMAZSA → compaction'da SIL
- Celiskili lesson bulunursa (ayni hata, farkli fix):
  → Daha yeni olani TUT, eskisine `[SUPERSEDED by tarih]` ekle
  → Eski kaydi compaction'da SIL
- Compaction sirasinda `[L]` + 3 session gecmis → SIFIRLA (otomatik temizlik)
- Confidence OLMADAN lesson yazma = PROTOCOL VIOLATION

SIRA 2 — Skill Metadata (aktif skill varsa):
┌─────────────────────────────────────────────────────────────────────────┐
│ ~/.claude/skills/{skill}/SKILL.md   │ Guncelle:                        │
│                                     │ - lesson_count: {yeni sayi}      │
│                                     │ - last_updated: {bugunun tarihi} │
│                                     │ - maturity_score: (5 lesson'da   │
│                                     │   1 yeniden degerlendir)         │
│                                     │ - Bilinen Sorunlar listesi       │
│                                     │   (yeni bug varsa ekle)          │
│                                     │                                  │
│ DESYNC KORUMA: SKILL.md'yi          │ guncellemeden ONCE mevcut        │
│ lesson_count'u oku. Phase 3 Post-   │ Execution bu session'da ZATEN   │
│ guncelledi ise SIRA 2'yi ATLA.      │ Kontrol: lesson_count session    │
│ basindakinden buyuk mu?             │                                  │
└─────────────────────────────────────┴───────────────────────────────────┘

SIRA 3 — Session Handoff Dosyasi (aktif skill varsa):
┌─────────────────────────────────────────────────────────────────────────┐
│ ~/.claude/skills/{skill}/           │ Session sonuc bolumu EKLE:       │
│   SESSION-HANDOFF.md                │ - Tarih + session ozeti          │
│                                     │ - Tamamlanan gorevler            │
│                                     │ - Acik kalan gorevler            │
│                                     │ - Alinan kararlar                │
│                                     │ - Sonraki adimlar                │
│                                     │                                  │
│ NOT: Bu HAZIRLIK 3'teki guncelleme  │ session BITISINDE yapilir.       │
│ Prompt ADIM 9'daki "SESSION-HANDOFF │ guncelle" ise sonraki session    │
│ SIRASINDA her gorev sonrasi yapilir.│ Ikisi FARKLI zamanlamada calisir.│
└─────────────────────────────────────┴───────────────────────────────────┘

SIRA 4 — Kalici Hafiza:
┌─────────────────────────────────────────────────────────────────────────┐
│ ~/.claude/projects/{project}/       │ KALICI ogrenmeler varsa ekle:    │
│   memory/MEMORY.md                  │ - Yeni mimari kararlar           │
│                                     │ - Yeni kullanici tercihleri      │
│                                     │ - Yeni dosya konumlari           │
│                                     │ - Yeni servis bilgileri          │
│                                     │ GECICI bilgi yazma (task state,  │
│                                     │ in-progress is — bunlar prompt'a │
│                                     │ gider, MEMORY'ye degil)          │
└─────────────────────────────────────┴───────────────────────────────────┘

SIRA 5 — Proje Dosyalari (GSD aktifse):
┌─────────────────────────────────────────────────────────────────────────┐
│ {project}/.planning/PROJECT.md      │ Phase/milestone durumu guncelle  │
│ {project}/.planning/roadmap.md      │ Tamamlanan phase'leri isaretle   │
│ {project}/.planning/PLAN.md         │ Acik task'lari guncelle          │
└─────────────────────────────────────┴───────────────────────────────────┘

ONEMLI KURALLAR:
- Her dosya guncelleme Edit ile yapilir, ASLA Write (ustune yazma riski)
- Dosya yoksa ATLA, olusturma
- Session'da o dosyayla ilgili yeni bilgi yoksa ATLA
- Her guncelleme sonrasi 1 satirlik log: "errors.md +3 entry eklendi"
- Toplam guncelleme sayisini SAY ve ADIM 4'te raporla
```

```
HAZIRLIK 4: BUTUNLUK KONTROLU

Dosya guncellemeleri tamamlandiktan sonra:
- Guncellenen dosya sayisi: {N}
- Eklenen lesson sayisi: errors +{X}, golden +{Y}, edge +{Z}
- MEMORY.md guncellendi mi: Evet/Hayir
- SESSION-HANDOFF.md guncellendi mi: Evet/Hayir
- Celiskili bilgi var mi: (ayni hata farkli fix ile kaydedildiyse)

Bu ozeti kullaniciya 1-2 satirla bildir:
"7 dosya guncellendi (errors +3, golden +1, MEMORY +2 entry). Simdi gecis
 promptunu hazirliyorum."
```

```
HAZIRLIK 5: Asagidaki PROMPT YAPISI'na gore SOMUT, DOLDURULMUS gecis promptu uret.
  {{placeholder}} YASAK. Gercek verilerle doldur.
  Prompt icinde ADIM 3'te guncellenen dosyalarin HEPSINI "zorunlu okuma" listesine koy.
  (Recursive baglanti: sonraki session bu guncellenmis dosyalari OKUYACAK)
  ADIM 4.5'i HAZIRLIK 2'deki TaskList snapshot'indan doldur (in_progress + pending task'lar)
```

```
HAZIRLIK 6: Prompt'u kullaniciya goster + /tmp/session-handoff-{tarih}.md'ye kaydet.

  Kayit sonrasi kullaniciya:
  "Gecis promptu hazir. /tmp/session-handoff-{tarih}.md dosyasina da kaydettim.
   Yeni session'a yapistirdiginizda tum tetikleyiciler otomatik aktif olacak."
```

### TOKEN CIMRILIGI YASAGI (TEKRAR TEKRAR VURGULANIYOR)

```
BU PROMPT URETILIRKEN:
- ASLA kisaltma yapma
- ASLA "vb.", "vs.", "..." ile gecistirme
- ASLA "dosyalari oku" gibi lazy referans verme
- ASLA ozet/summary moduna gecme
- ASLA "daha once bahsedildigi gibi" deme — TEKRAR YAZ

NEDEN: Sonraki session SIFIR context ile basliyor.
O Claude senin bildiklerini BILMIYOR.
Her dosya yolu, her sayi, her komut, her karar TEKRAR yazilmali.

Bu prompt 2000+ token olabilir — SORUN DEGIL.
Bu prompt 5000+ token olabilir — HALA SORUN DEGIL.
Kisa prompt = sonraki session'da kayip context = verimsiz calisma = GERCEK israf.
```

---

## MANUEL TETIKLEME (Ek — kullanici acikca isterse)

Otomatik tetiklemeye ek olarak, kullanici su ifadelerden birini kullandiginda
da AYNI protokol baslar:

1. "sonraki chat icin prompt hazirla"
2. "gecis promptu olustur"
3. "handoff yap"
4. "session bitiyor, kaydet"
5. "bunu sonraki session'a aktar"

---

## PROMPT YAPISI (Bu sirada, bu bolumlerle uret)

Sonraki session'a yapistilacak prompt su bolumlerden OLUSMALI:

---

### BOLUM 1: BASLIK + KIMLIK

```
─── HANDOFF META ───
trigger: {HAZIRLIK 2'den: explicit_user | milestone_complete | mid_task}
session: {kacinci handoff} | protocol: v{versiyon}
active_skills: {HAZIRLIK 2'den: skill listesi veya []}
pipeline: {HAZIRLIK 4'ten: complete | partial} ({N} dosya)
lessons: errors+{X}, golden+{Y}, edge+{Z}
coverage: {gorev turleri: bridge, gsd, rules-edit vb.}
─── END META ───

YENi SESSION BASLANGICI — {Proje Adi} / {Session Amaci}
Bu session onceki uzun bir oturumun devamidir.
Asagidaki adimlari SIRASYLA uygula — bolum atlama, kisaltma, token tasarrufu YASAK.
NOT: Bu prompt YENI (sifir-context) session icin tasarlandi. Eger mevcut bir
session'i resume ediyorsan (claude --resume), ADIM 1-2 atla, ADIM 3'ten basla.
```

HAZIRLIK 6'da /tmp/ dosyasina kaydederken YAML frontmatter EKLE:
```yaml
---
generated_at: {tarih}
trigger_reason: {explicit_user | milestone_complete | mid_task}
protocol_version: v{versiyon}
session_number: {N}
active_skills: [{skill listesi}]
pipeline_status: {complete | partial}
files_updated: {N}
lessons_added: {errors: N, golden: N, edge: N}
coverage_scope: [{gorev turleri}]
---
```
Bu frontmatter SADECE /tmp/ dosyasinda — prompt'a yapistirilan kisimda HANDOFF META blogu yeterli.

---

### BOLUM 2: AKILLI CONTEXT YUKLEME (MANIFEST-BASED)

Sonraki session'in ADIM 1'ini olustur. 3 katmanli manifest kullan:

**KATMAN 1 — AUTO-LOAD (Read GEREKSIZ, zaten context'te):**
~/.claude/rules/*.md (9 dosya) ve MEMORY.md otomatik yuklenir.
Bu dosyalari ADIM 1'e "Oku" olarak YAZMA — token israfi.
Sadece session'da DEGISEN auto-load dosyalarini BILGI NOTU olarak ekle:

```
ADIM 1: AKILLI CONTEXT YUKLEME

Once HANDOFF META blogunu oku (prompt basinda).
- active_skills bos ise → KATMAN 3'te skill dosyalarini ATLA
- trigger: milestone_complete ise → context yuklemeyi hizlandir, temiz gecis
- trigger: mid_task ise → devam eden isi oncelikle yukle
- pipeline: partial ise → lesson dosyalarina GUVENME, eski olabilir

─── AUTO-LOADED (zaten context'inde — Read YAPMA, dikkat et) ───
| Dosya | Bu Session'da Degisen |
|-------|----------------------|
| {dosya_adi} | {ne degisti — 1 satir} |
(Degismeyen auto-load dosyalarini LISTELEME)
```

**KATMAN 2 — ZORUNLU Read (her gorev turunde):**
Handoff pipeline'da guncellenen NON-AUTO-LOAD dosyalar.
Bunlar context'te YOK — MUTLAKA Read ile okunmali:
- Skill lessons (errors.md, golden-paths.md, edge-cases.md)
- SESSION-HANDOFF.md, SKILL.md
- _shared/common-errors.md
Sadece GERCEKTEN guncellenen dosyalari listele.

```
─── ZORUNLU OKU (context'inde YOK) ───
1. {tam_yol}
   → {odak bolum} ({neden onemli})
   → [HANDOFF'TA GUNCELLENDI: {ne eklendi}]
```

**KATMAN 3 — ON-DEMAND Read (gorev turune gore):**
ADIM 3'teki master goal'den gorev turunu tespit et.
Asagidaki tabloyu kullanarak sadece ilgili dosyalari listele:

| Gorev Turu | Tetikleyen Keyword'ler | On-Demand Dosyalar |
|------------|----------------------|-------------------|
| bridge | openclaw, bridge, gateway, session takeover | `skills/openclaw-manage/{SKILL,SESSION-HANDOFF,lessons/*}.md` |
| gsd | phase, milestone, roadmap, plan, .planning | `{project}/.planning/{PROJECT,roadmap,PLAN}.md` |
| skill-work | skill, lesson, golden-path, maturity | `skills/{aktif-skill}/{SKILL,SESSION-HANDOFF,lessons/*}.md` |
| debug | bug, hata, fix, error, debug | `skills/{aktif-skill}/lessons/errors.md`, `_shared/common-errors.md` |
| ftth | fiber, odoo, ftth, module | `skills/ftth-*/SKILL.md`, `skills/odoo-*/lessons/errors.md` |
| rules-edit | handoff, protocol, rules, kural | (tumu auto-load — ek Read YOK) |
| genel | (fallback) | `_shared/common-errors.md` |

Birden fazla tur eslesebilir — her iki turun dosyalarini listele.
KATMAN 2'de zaten listelenen dosyalari TEKRARLAMA.

```
─── ON-DEMAND OKU (gorev turu: {tespit_edilen_tur}) ───
1. {tam_yol}
   → {neden bu gorev turunde gerekli}
(Sadece tespit edilen ture ait, KATMAN 2'de olmayan dosyalar)
```

**KURALLAR:**
- Tam dosya yolu ZORUNLU — lazy referans ("skill dosyalarini oku") YASAK.
- KATMAN 2 + KATMAN 3 toplamda 10+ dosya oluyorsa: manifest YANLIS.
  Gozden gecir — auto-load dosyasi KATMAN 2/3'e kacmis olabilir.
- RECURSIVE BAGLANTI hala gecerli: yaz → listele → sonraki oku → calis → yaz → ...

---

### BOLUM 3: ALTYAPI DURUM KONTROLU

Calisan servislerin health check komutlarini yaz. Kopyala-yapistir hazir.

```
ADIM 2: DURUM KONTROLU

# {Servis 1}
{tam_komut}
# Beklenen: {beklenen cikti}
# DOWN ise: {recovery komutu — KULLANICIYA SUN, Claude CALISTIRMAZ}

# {Servis 2}
...

KISITLAMA: Recovery komutlari kill/pkill/killall ICERMEZ.
Servis restart gerekiyorsa non-destructive alternatif sun (systemctl restart, docker restart vb.)
veya "kullanici manuel restart etmeli" yaz. session-safety.md Iron Law gecerli.

Bilinen anomaliler (gormezden gel):
- {anomali} → bilinen bug #{N}, calismayi ETKILEMEZ
```

---

### BOLUM 4: MASTER SCOPE

```
ADIM 3: BU SESSION'IN AMACI

Genel baglam: {3-5 cumle — proje ne, onceki session'da ne yapildi, bu session
ne yapacak, neden onemli}

Bu session'in TEK ANA HEDEFI:
{Tek cumle, net, olculebilir}

Scope sinirlar:
ICINDE: {madde madde — ne yapilacak}
DISINDA: {madde madde — ne YAPILMAYACAK, zaman kaybi olan seyler}
```

---

### BOLUM 5: STRATEJI & EXECUTION PLANI (KATMANLI)

Yapilacak isi katmanlara bol. Her katman icin:
- Amaci (neden bu katman var)
- Somut gorev (ne yapilacak)
- Basari kriteri (ne oldugunda "tamam" denecek)
- Belgelenecekler (ne kaydedilecek)

```
ADIM 4: KATMANLI STRATEJI

Her katman sirayla uygulanacak. Katman 1 bitmeden Katman 2'ye GECME.

KATMAN 1 — {Isim} ({Kisa aciklama})
  Amaci: {neden}
  Gorev: {somut, yapilabilir gorev tanimi}
  Basari kriteri: {olculebilir}
  Belgelenecekler: {ne kaydedilecek, nereye}

KATMAN 2 — {Isim} ({Kisa aciklama})
  ...

KATMAN N — {Isim} (Paralel agent'larla)
  Paralel agent sayisi: {N}
  Agent dagilimlari:
    Agent A — {gorev} — {dosya scope'u}
    Agent B — {gorev} — {dosya scope'u}
    Agent C — {gorev} — {dosya scope'u}
```

---

### BOLUM 5.5: TASK STATE SERIALIZATION

Handoff promptunda devam eden gorevleri metin olarak tasir. TaskCreate in-memory oldugu icin
session arasi kaybolur — bu bolum state'i TEXT olarak serialize eder.

```
ADIM 4.5: DEVAM EDEN GOREVLER (TaskList Snapshot)

Onceki session'dan kalan TAMAMLANMAMIŞ gorevler:

| # | Subject | Status | BlockedBy | Description |
|---|---------|--------|-----------|-------------|
{Her in_progress ve pending task icin 1 satir — HAZIRLIK 2'deki TaskList snapshot'indan al}

Tamamlanan gorev sayisi: {N} (referans icin, yeniden OLUSTURMA)

SONRAKI SESSION KURALI:
Session baslarken (ADIM 1'den sonra) bu tabloyu oku ve:
1. Her satir icin TaskCreate cagir (subject + description koru)
2. addBlockedBy ile dependency'leri kur
3. in_progress olanlari hemen TaskUpdate(status: in_progress) yap
4. Tamamlanan task'lari OLUSTURMA — sadece bilgi amacli

NEDEN: TaskCreate in-memory, session arasi KAYBOLUR.
Bu tablo state'i TEXT olarak tasir → sonraki session yeniden olusturur.
```

---

### BOLUM 6: OTONOM ORCHESTRATION TETiKLEYiCiLERi ⚡

**BU BOLUM KRiTiK.** Sonraki session boyunca AKTIF kalan tetikleyiciler.
Claude bunlari okudugunda otomatik olarak uygulamaya baslar.

```
ADIM 5: SESSION BOYUNCA AKTiF TETiKLEYiCiLER

Asagidaki tetikleyiciler bu session BOYUNCA gecerlidir.
Kosul gerceklestiginde belirtilen aksiyonu OTONOM uygula — benden izin BEKLEME.

─── MAKRO SCOPE INJECTION (TUM SUB-AGENT'LAR ICIN ZORUNLU) ───

Her sub-agent spawn edildiginde Task prompt'una ADIM 3'teki ICINDE/DISINDA scope
sinirlarini INJECT ET. Format (autonomous-orchestration.md ile uyumlu):

SCOPE SINIRI:
ICINDE: {ADIM 3'ten kopyala}
DISINDA: {ADIM 3'ten kopyala}
Scope disi → DUR, koordinatore raporla.

Scope injection OLMADAN sub-agent spawn = PROTOCOL VIOLATION.

─── SUB-AGENT SPAWN TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| 3+ bagimsiz is akisi tespit edildi | Her akisa 1 agent spawn et |
| Arastirma + uygulama + dogrulama birlikte | Explore + general-purpose + code-reviewer paralel |
| 5+ dosya analizi gerekli | Explore agent spawn et |
| Test + kod ayni anda yazilacak | 2 paralel agent (TDD: test agent ONCE baslar) |
| Entegrasyon noktasi test edilecek | general-purpose + debugger paralel |
| Security/performance/lint taramasi | infra-guardian veya code-reviewer spawn et |
| Ben "hizli", "paralel", "swarm", "agent spawn et", "multi-agent", "otonom yap" dersem | Complexity mapping'e gore 3-10 agent |
| Buyuk codebase analizi + coklu raporlama | Explore agent spawn et |
| Katman N'de "paralel" yaziyorsa | Belirtilen sayida agent spawn et |
| ADIM 4'te "Paralel agent" yazan katman var | Katmandaki agent sayisi + gorev dagilimindan TaskCreate + Task spawn |
| ADIM 4'te 3+ katman ve her biri farkli uzmanlik gerektiriyor | Her katman icin specialist agent (Explore/general-purpose/code-reviewer) sec + scope inject et |
| Katman gecisi oldu ve sonraki katman paralel | Onceki katman task'larini completed yap, yeni katman task + agent spawn |
| Agent koordinasyonu tamamlandi | Success Contract uret: Spawned/Completed/Results Collected/Missing/Status |

Wave pattern: 3-4 agent → 2+2 | 5-8 → 3+3+2 | 9-10 → 4+4+2
Complexity: Medium 3-4 | High 5-8 (default 5) | Very High 9-10 (default 10)
Guardrails: Synthetic task ID YASAK. ToolSearch path'ine dayanma. Hata → tek tek retry (max 3).
Real agentId kaydet. Tum ciktilar toplanmadiysa PARTIAL raporla, SUCCESS deme.

─── TASK LIST OLUSTURMA TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| Handoff promptunda ADIM 4.5 tablosu var | Tabloyu oku → her satir icin TaskCreate + addBlockedBy + status kur |
| 2+ gorev listelendi (benim tarafimdan veya strateji bolumunde) | Her gorev → ayri TaskCreate |
| Gorev 3+ adim gerektiriyor | Adimlari task olarak olustur |
| Agent spawn edilecek ve 3+ agent var | Her agent gorevi → task, agent → owner |
| Bug fix baslatildi (3+ hipotez) | Her hipotez → ayri task |
| Validation senaryolari belirlendi | Her senaryo → ayri task |
| Katman gecisi oldu | Yeni katmanin task'larini olustur |

Her TaskCreate'te: subject (imperatif), description (detayli), activeForm (present continuous).
Bagimliliklar: addBlockedBy/addBlocks ile hemen kur.

─── SUB-TASK BOLME TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| Task description 5+ adim iceriyor | Her adim → sub-task |
| Task birden fazla dosya etkiliyor | Dosya grubu basina sub-task |
| Task hem test hem impl gerektiriyor | Test + impl + verify = 3 sub-task |
| Task birden fazla agent'a atanabilir | Agent basina sub-task |
| Task'ta birden fazla acceptance criteria var | Her kriter → dogrulama sub-task'i |

─── SEYTANIN AVUKATI SPAWN TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| Mimari karar verilecek (pattern, teknoloji, tasarim) | Karar VERILMEDEN ONCE devil's advocate agent spawn |
| 5+ dosya etkileyen degisiklik planlaniyor | Plan ONAYLANMADAN ONCE spawn |
| Guvenlik degisikligi (auth, crypto, API key, CORS) | Kod YAZILMADAN ONCE spawn |
| Breaking change veya backward-incompatible degisiklik | Commit ONCESI spawn |
| Ben "emin misin?", "risk var mi?" dersem | HEMEN spawn |
| Trade-off karari (hiz vs guvenlik, basitlik vs esneklik) | Karar aninda spawn |

Devil's advocate agent'in gorevi: 3+ zayif nokta bul, 2+ alternatif sun, 1-10 risk puani ver.
Risk 1-3 → DUR kullaniciya sor | 4-6 → alternatiflerle sun | 7-10 → devam et, logla.

─── VALIDATION TEST CASE TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| Yeni endpoint/API eklendi | curl/httpie ile validation test case yaz ve CALISTIR |
| Yeni fonksiyon/method yazildi | Unit test yaz ve CALISTIR |
| Mevcut davranis degistirildi | Eski + yeni davranis regression testi |
| Config/environment degisti | Oncesi/sonrasi karsilastirma testi |
| Entegrasyon noktasi eklendi/degisti | End-to-end akis testi |
| Ben "test et", "dogrula", "calisiyor mu?" dersem | istenen seviyede test |

Her test icin: Komut CALISTIR → ciktiyi OKU → SONRA "pass/fail" iddia et.
"Should work" = GECERSIZ. Kanit goster.

─── PRODUCTION-GRADE TEST TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| Feature "tamamlandi" isaretlenecek | Production-grade test suite yaz |
| Ben "production-ready", "ship it", "deploy" dersem | Full test coverage |
| Phase verify asamasina gelindi | Phase UAT testleri |
| Milestone tamamlaniyor | Regression suite |
| Breaking change yapildi | Migration + backward compat testleri |
| 3+ ardisik basarisiz fix/task | DUR, kullaniciya raporla, /custom:deep-debug spawn et |

Production test = happy path + 2 error path + 1 edge case + 1 negative + idempotency + isolation.
TDD gecerli: RED → GREEN → REFACTOR.
GSD aktifse: /custom:tdd-enforce (plan sonrasi), /custom:deep-debug (spot-check fail), /custom:validate-gap (milestone sonrasi).
Debug agent spawn edildiginde: troubleshooting.md DOING/EXPECT/IF WRONG pattern'i inject et.

─── CODE REVIEW TETiKLEYiCiLERi ───

| Kosul | Aksiyon |
|-------|---------|
| 50+ satir kod degisikligi | code-reviewer agent spawn |
| Guvenlik-sensitif dosya degisti | code-reviewer + infra-guardian spawn |
| API endpoint eklendi/degisti | API review agent spawn |
| Infrastructure kodu degisti | infra-guardian spawn |

─── KALITE KAPISI TETiKLEYiCiLERi ───

Her asama gecisinde (plan → impl → test → complete) kalite kapisi UYGULA:

Plan → Impl: Plan var mi + onaylandi mi + DA calisti mi + test stratejisi var mi
Impl → Test: Kod yazildi mi + lint/format gecti mi + review yapildi mi
Test → Complete: Tum testler PASS mi + evidence var mi + lesson guncellendi mi
Complete → Deploy: Smoke PASS mi + rollback plani var mi + kullanici onay verdi mi

Kapiyi gecemeyen is sonraki asamaya ILERLEYEMEZ.
```

---

### BOLUM 7: PARALEL BUG FIX / EK GOREVLER (varsa)

Onceki session'dan kalan acik buglar veya validation'a paralel yurutulecek isler varsa
tablo halinde yaz.

```
ADIM 6: PARALEL YURUTULECEK ISLER (Validation blokluyorsa)

| Task # | Dosya | Fix/Gorev | Bagimsiz mi? | Priority |
|--------|-------|-----------|-------------|----------|
| {#N}   | {dosya} | {aciklama} | Evet/Hayir | P0/P1/P2 |
```

---

### BOLUM 8: GUVENLIK / KISITLAR NOTU

Bilinen guvenlik aciklari, ertelenmis kararlar, yapilmayacak seyler.
Session sirasinda zaman kaybini onler.

```
ADIM 7: GUVENLIK NOTU (EYLEM YOK, BiLGi AMACLI)

Su an {ortam aciklamasi}. Asagidaki konular bilerek ertelendi, bu session'da
KONUSMAYA BiLE ACMA:
- {ertelenmis karar/acik 1}
- {ertelenmis karar/acik 2}

BRIDGE NOTU (bridge session'larinda gecerli):
- agentToAgent.enabled: false — Task tool agent spawn calisir ama bridge routing OLMAZ
- Bridge-spawned session'larda MCP tools yok — MCP-dependent skill'ler CALISMAZ
- HAZIRLIK 3 pipeline sirasinda bridge mesaj gonderebilir — gerekirse once /pause
```

---

### BOLUM 9: REFERANSLAR

Kritik sayilar, dosya konumlari, endpoint'ler, performans baseline'lari.
Sonraki session'daki Claude'un sik ihtiyac duyacagi bilgiler.

```
ADIM 8: REFERANSLAR

Kritik dosyalar:
{dosya_yolu_1} → {icerik}
{dosya_yolu_2} → {icerik}

Endpoint'ler:
{METHOD} {path} → {aciklama} ({auth durumu})

Performans baseline (varsa):
{metrik}: {deger} ({kaynak})

Bilinen acik buglar:
#{N} {aciklama} — {dosya} — {etki}

Lesson dosyalari:
{skill_lessons_errors}   ({N} entry)
{skill_lessons_golden}   ({N} entry)
{skill_lessons_edge}     ({N} entry)

MCP VERI KURALI: Endpoint, metrik, bug sayisi gibi veriler MCP tool'dan
geldiyse [kaynak: {tool_adi}, {zaman}] ile etiketle. Hafizadan uretme —
mcp-anti-hallucination.md Iron Law gecerli. Read/Grep ile okunan dosya
verileri (lesson count vb.) MCP kapsaminda DEGIL, bunlar guvenli.
```

---

### BOLUM 10: BASARININ TANIMI

Session sonunda cevaplanmis olmasi GEREKEN sorular + kabul edilebilir kanit.

```
ADIM 9: BASARININ TANIMI

Session sonunda su sorulara KANITA DAYALI cevabin olmali:

| Soru | Kabul Edilebilir Kanit |
|------|------------------------|
| {soru_1} | {somut kanit — komut ciktisi, log satiri, test sonucu} |
| {soru_2} | {somut kanit} |
| ...  | ... |

"Should work" GECERSIZ kanit. Komutu CALISTIR, ciktiyi goster, SONRA iddia et.

Her test/gorev sonrasi:
- errors.md yeni bulgu varsa guncelle (Edit, ASLA Write)
- golden-paths.md basarili pattern varsa ekle
- SESSION-HANDOFF.md sonuca bolum ekle (Edit, ASLA Write)
- SKILL.md versiyon + lesson_count guncelle
```

---

### BOLUM 11: ACIK KARARLAR

Onceki session'dan kalan, bu session'da cevaplanmasi gereken sorular.

```
ADIM 10: ACIK KARARLAR

Onceki session'da su mimari/teknik soruyu tespit ettik:
{sorunun detayli aciklamasi}

Bu session'in sonunda karar vermemiz gerekiyor:
- Secenek A: {aciklama + trade-off'lar}
- Secenek B: {aciklama + trade-off'lar}
- Secenek C: {varsa}

Bu soruya net cevap verirken VERi URET. Her secenegin trade-off'larini OLC.
```

---

### BOLUM 12: BASLA KOMUTU

```
───────────────────────────────────────────────
BASLA! Context'i yukle (ADIM 1), servisleri kontrol et (ADIM 2),
sonra master goal'e (ADIM 3) gore calis.
ADIM 5'teki tetikleyiciler SESSION BOYUNCA AKTIF — surekli uygula.
Token tasarrufu YAPMA. Detayli, kapsamli, otonom calis.
───────────────────────────────────────────────
```

---

## PROMPT URETME KURALLARI

Gecis promptu uretirken su kurallara UY:

1. **SOMUT OL:** {{placeholder}} YASAK. Gercek dosya yollari, gercek komutlar, gercek sayilar.
2. **KOPYA-YAPISTIR HAZIR:** Komutlar direkt calistirilabilir olmali. "su komutu calistir" degil, komutu GOSTER.
3. **LAZY REFERANS YASAK:** "Skill dosyalarini oku" degil, her dosyanin TAM YOLUNU ve ODAK BOLUMUNU yaz.
4. **SAYI VER:** "Birkac bug" degil, "6 bug, #11-#16". "Lesson dosyalari" degil, "errors.md (63 entry)".
5. **TOKEN CIMRILIGI YASAK:** Kisa tutma. Sonraki Claude SIFIR context ile basliyor. Her detay lazim.
6. **TETiKLEYiCiLER EMBED:** Bolum 6'daki tetikleyici tablolari HER ZAMAN dahil et. Bunlar prompt'un KALBI.
7. **ONCEKI SESSION STATE:** Tam olarak neredeyiz? Ne tamamlandi, ne kaldi, ne bloklandi? NET yaz.
8. **KANIT STANDARDI:** "Calisiyor" degil, "curl output X dondu" seviyesinde kanit talep et.

## ANTI-PATTERN'LER (YAPMA)

| Anti-Pattern | Dogru Yaklasim |
|-------------|----------------|
| "Dosyalari oku ve devam et" | Her dosyanin tam yolunu + odak bolumunu yaz |
| "Testleri calistir" | Hangi testler, hangi komut, beklenen cikti ne |
| "Agent spawn et gerekirse" | Tetikleyici tablosu ile OTOMATIK spawn kurali koy |
| "Task list olustur" | Tetikleyici tablosu ile OTOMATIK olusturma kurali koy |
| "Gerekirse devil's advocate" | Tetikleyici tablosu ile OTOMATIK spawn kurali koy |
| Tetikleyicileri atlayip sadece gorev yazmak | Tetikleyiciler = prompt'un degeri. HER ZAMAN dahil et |
| Kisa/ozet gecis promptu | Uzun, detayli, her sey acik |
| Dosya guncellemeden prompt uretmek | ADIM 3 pipeline ZORUNLU — once dosyalar, sonra prompt |
| Write ile lesson/handoff guncellemek | ASLA Write, HER ZAMAN Edit (ustune yazma riski) |
| Lesson'a prose yazmak | Tablo satiri formati ZORUNLU |
| Dosya yoksa olusturmak | Handoff pipeline dosya OLUSTURMAZ, sadece mevcut gunceller |
| MEMORY.md'ye gecici bilgi yazmak | Gecici = prompt'a yaz. Kalici = MEMORY'ye yaz |
| Guncelleme yapmadan "guncellendi" demek | Her Edit sonrasi 1 satirlik log: "+N entry eklendi" |

---

*Version: 2.5.0 — Last updated: 2026-02-28 (Task State Serialization, TaskList Generator, Specialist Agent Generator)*

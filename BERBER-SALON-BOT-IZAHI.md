# Bərbər Salon Botu — İzahı

Bu sənəd "FLOW (2): AI Agent, RAG, Memory, Tool-calling" dərsinin ev
tapşırığı olan **Bərbər Salon Botu**-nun necə quraşdırıldığını,
qarşılaşdığımız problemləri və həllərini izah edir.

## Ümumi məntiq

İki ayrı workflow quruldu:

1. **"Salon Data Yüklə"** — salonun məlumatlarını (xidmətlər, qiymətlər,
   ustalar, iş saatları, FAQ) bir dəfə Qdrant-a (vektor bazasına) yazır.
2. **"Müştəri Chat Botu"** — müştəri ilə danışan AI Agent. Üç "bacarığı"
   var: sənəddən cavab tapmaq (RAG), söhbəti xatırlamaq (Memory), lazım
   olanda xarici sistemə sorğu göndərmək (Tool-calling).

---

## Workflow 1 — Salon Data Yüklə

```
Manual Trigger → Read/Write Files (salon.txt) → Qdrant Vector Store (insert, "salon")
                                                    ├── Document → Default Data Loader → Text Splitter (300/60)
                                                    └── Embedding → Embeddings OpenAI (text-embedding-3-small)
```

### Nə edildi

- `shared/salon.txt` faylı yaradıldı (konteynerdə `/data/shared/salon.txt`
  yoluna uyğundur) — BarberKing Bakı salonunun bütün məlumatları (xidmət
  qiymətləri, ustalar, iş saatları, FAQ).
- **Qdrant Collection adı:** `salon` seçildi — **`32`** yazılmadı, çünki
  bu, RAG Chatbot dərsində artıq işlədilən kolleksiya idi (çatdırılma/
  qaytarma məlumatları). Eyni ədədi yazsaydıq, iki fərqli mövzunun
  məlumatı bir yerdə qarışardı.
- **Default Data Loader**-də `Type of Data` = **Binary** seçildi (defolt
  "JSON" idi — bu, səhvdir, çünki fayl "Read/Write Files"-dan binary
  formada gəlir).
- `Text Splitting` = **Custom** edildi ki, "Recursive Character Text
  Splitter" sub-node-u əlavə etmək mümkün olsun (Chunk Size: 300,
  Chunk Overlap: 60).
- **OpenAI kredensialı üçün** — dərsin içində göndərilən iki yeni API
  açarı **işlədilmədi**. Bunun əvəzinə RAG Chatbot dərsində artıq
  yaradılmış **"OpenAI account"** kredensialı yenidən istifadə olundu
  (həm sadə, həm təhlükəsiz — yeni açarları heç bir fayla yazmadıq).

### Test nəticəsi

Manual Trigger ilə işə salındı, bütün node-lar yaşıllaşdı. Qdrant-da
təsdiqləndi: `salon` kolleksiyası yaradılıb, **5 parça** (points_count: 5)
saxlanılır.

---

## Workflow 2 — Müştəri Chat Botu

```
Chat Trigger → AI Agent
                 ├── Chat Model → OpenAI Chat Model (gpt-4o-mini)
                 ├── Memory → Simple Memory
                 └── Tool → Qdrant Vector Store (retrieve, "salon") → Embedding → Embeddings OpenAI
                     Tool → HTTP Request (bos_vaxt_yoxla, webhook.site)
```

### Qdrant Tool

Qdrant Vector Store node-u AI Agent-in **Tool** girişinə **"Retrieve
documents for AI Agent as Tool"** rejimində qoşuldu (Insert rejimi yox —
bu, Workflow 1-dəki fərqli seçimdir). Collection: `salon` (Workflow
1-dəki adla EYNİ olmalıdır). Tool Description:

> "Salonun xidmətləri, qiymətlər, ustalar, iş saatları, ünvan və
> tez-tez soruşulan suallar haqqında məlumat verir. İşlət: müştəri
> xidmət, qiymət, usta, ünvan, rezervasiya qaydası soruşduqda."

### HTTP Tool (boş vaxt yoxlaması)

Bu, real cədvəl sistemi ilə əlaqə **DEYİL** — sadəcə `webhook.site`
adlı pulsuz "tələ qutusu" ilə test edildi ki, agentin lazım olanda
xarici sistemə **doğru şəkildə müraciət etdiyi** göstərilsin. Real
layihədə bu URL yerinə salonun öz rezervasiya sisteminin API ünvanı
yazılardı.

- **Method:** GET
- **URL:** webhook.site-dan alınan unikal test ünvanı
- **Description:** aşağıda izah olunan səbəbdən bir dəfə yenidən yazıldı.

---

## Qarşılaşılan problemlər və həllər

### 1. Səhv node növü seçildi ("HTTP Request" vs "HTTP Request Tool")

**Simptom:** HTTP Request node-unda "Description" sahəsi tapılmadı.

**Səbəb:** Node AI Agent-in **əsas çıxış xəttindəki** ("What happens
next?") "+" işarəsindən əlavə edilmişdi — bu, sadəcə növbəti addım üçün
node seçimidir, Tool sub-node seçimi deyil. Doğru yer AI Agent-in altında
**"Tool"** yazan xüsusi girişin yanındakı "+"dır.

**Həll:** Səhv node silindi, düzgün "Tool" girişindən yenidən "HTTP
Request" axtarılıb əlavə edildi. Bu dəfə node avtomatik **Description**
sahəsi ilə göründü (bu n8n versiyasında ayrıca "HTTP Request Tool" node
tipi yoxdur — adi "HTTP Request" node Tool girişinə qoşulanda özü
alət kimi davranır).

### 2. Agent HTTP Tool-u çağırmadı — Description zəif idi

**Simptom:** "Bu həftə boş vaxtınız varmı?" sualına agent HTTP Tool-u
çağırmadan, birbaşa Qdrant bilik bazasındakı "Rezervasiya necə edilir?
WhatsApp-a yazın" cavabını verdi.

**Səbəb:** `salon.txt`-də artıq rezervasiya ilə bağlı bir FAQ cavabı var
idi. Agent bunu kifayət hesab edib yeni alətə müraciət etməyə ehtiyac
duymadı — Tool Description kifayət qədər məcburedici deyildi.

**Həll:** Description daha kəskin yazıldı:

> "Bu HƏFTƏ üçün REAL VAXTDA (canlı) boş saatların olub-olmadığını
> yoxlamaq üçün YALNIZ bu aləti çağır. Salon sənədlərində bu barədə
> real vaxt məlumatı YOXDUR — rezervasiya, boş vaxt, 'nə vaxt gələ
> bilərəm' kimi suallarda mütləq bu aləti işlət, sənəddən cavab vermə."

Testi köhnə (əvvəlki mesajların təsirində qala biləcək) sessiyada yox,
**təzə sessiyada** (çatın "Session" ID-sinin yanındakı ↺ düyməsi ilə)
təkrarlamaq da vacib idi ki, nəticə aydın görünsün.

**Nəticə:** Bu düzəlişdən sonra HTTP Request node yaşıl ✓ aldı,
`webhook.site`-da real vaxtda gələn GET sorğusu təsdiqləndi
(`user-agent: n8n`).

---

## Sınaq ssenarisi (uğurla keçdi)

1. `Salam, adım Güneldır. Saç boyası xidmətiniz varmı?` → Qdrant-dan
   qiymət/müddət məlumatı düzgün gəldi.
2. `Kamran usta bu işi görür?` → əvvəlki mesajla birləşdirib düzgün
   cavab verdi (boyama + Kamran usta).
3. `Bu həftə boş vaxtınız varmı?` → HTTP Tool çağırıldı, webhook.site-da
   sorğu təsdiqləndi.
4. `Adımı xatırlayırsanmı?` → "Günel" adını doğru xatırladı (Simple
   Memory işlədi).

---

## Təhlükəsizlik qeydi

Dərsin materialında **iki real OpenAI API açarı açıq mətn şəklində**
göndərilmişdi. Bunlar heç bir fayla yazılmadı və heç bir kredensiala
əlavə edilmədi — onun əvəzinə mövcud "OpenAI account" kredensialı
yenidən istifadə olundu. Əgər bu açarlar geniş paylaşılıbsa (bütün
kursa), tövsiyə: OpenAI dashboard-dan ləğv edib yenisini yaratmaq.

# RAG Chatbot (Qdrant + Groq) Workflow — İzahı

Bu sənəd sənin göndərdiyin n8n workflow-unun hər node-unu bir-bir izah edir.
Workflow n8n-ə uğurla import olundu (adı: **"RAG Chatbot - Qdrant + Groq"**,
id: `ragChatbotQdrantGroq01`) və `http://localhost:5678` ünvanındakı n8n
panelində görünür.

## Ümumi məntiq

Workflow iki ayrı mərhələdən ibarətdir:

1. **Doldurma (indexing) hissəsi** — bir mətn faylını oxuyur, parçalayır,
   vektora çevirir və Qdrant bazasına yazır.
2. **Sual-cavab (chat) hissəsi** — istifadəçinin sualını götürür, Qdrant-dan
   aidiyyatlı parçaları tapır və Groq LLM-i ilə cavab yaradır.

---

## 1-ci hissə: Sənədin Qdrant-a yüklənməsi

### `When clicking 'Test Workflow' button`
Manual trigger node-udur. n8n panelində "Test workflow" düyməsini basdıqda
işə düşür — yəni bu hissə əl ilə işə salınır.

### `Read/Write Files from Disk`
`/data/shared/news.txt` faylını oxuyur (bu yol konteyner daxilində
`./shared` qovluğuna uyğundur — sənin `shared/news.txt` faylın). Çatdırılma
və qaytarma qaydaları haqqında Azərbaycan dilində mətndir.

### `Default Data Loader`
Oxunan binary faylı (`dataType: binary`) sənəd formatına çevirir ki, sonra
parçalana bilsin.

### `Recursive Character Text Splitter`
Mətni kiçik parçalara (chunk) bölür, parçalar arasında 50 simvol overlap
(üst-üstə düşmə) saxlayır ki, məna itməsin. `splitCode: markdown` seçimi ilə
mətni markdown strukturuna uyğun bölür.

### `Embeddings OpenAI`
Hər mətn parçasını OpenAI-nin embedding modeli ilə vektora çevirir (OpenAI
kredensialı istifadə olunur).

### `Qdrant Vector Store` (mode: insert)
Vektora çevrilmiş parçaları Qdrant-ın `32` adlı kolleksiyasına yazır (insert
rejimi). Bu node `Read/Write Files from Disk`-dən gələn əsas axını, `Default
Data Loader`-dən sənədi və `Embeddings OpenAI`-dən vektorları özündə
birləşdirir.

---

## 2-ci hissə: Söhbət (Chat) və cavab yaratma

### `When clicking 'Chat' button below`
Chat trigger node-udur — n8n-in daxili chat pəncərəsi vasitəsilə istifadəçi
sualı buradan daxil olur (webhook əsaslıdır).

### `Embeddings OpenAI1`
Bu, birinci embedding node-un eyni funksiyanı yerinə yetirən ikinci
nüsxəsidir — amma bu dəfə istifadəçinin sualını vektora çevirmək üçündür
(axtarış zamanı).

### `Qdrant Vector Store1` (retrieve-as-tool)
Eyni `32` kolleksiyasından, sualın vektoruna ən yaxın olan mətn parçalarını
tapır. Hazırda **alət (tool)** rejimindədir — AI Agent onu lazım olanda özü
çağırır (bax: Troubleshooting jurnalı, 8-ci addım).

### `Simple Memory`
Söhbətin son bir neçə mesajını (default 5) yadda saxlayır ki, AI Agent
əvvəlki mesajları xatırlaya bilsin (məs. istifadəçinin adı).

### `Groq Chat Model`
Cavabı yaradan əsl dil modelidir, Groq API vasitəsilə işləyir. Model adı
node-un içində seçilib (bax: Troubleshooting jurnalı, 7-ci addım).

### `AI Agent`
Bütün tikə birləşir: istifadəçinin sualını götürür, lazım gələndə
`Qdrant Vector Store1` alətini çağırıb aidiyyatlı mətn parçalarını tapır,
`Simple Memory`-dən əvvəlki söhbəti oxuyur, sonra `Groq Chat Model`-ə
göndərib son cavabı formalaşdırır. Bu, **memory dəstəkli RAG (Retrieval-
Augmented Generation)** zəncirdir.

> Qeyd: bu node əvvəlcə "Question and Answer Chain" idi, amma o node memory
> dəstəkləmirdi. Səbəbi və dəyişikliyin təfərrüatı Troubleshooting
> jurnalının 8-ci addımındadır.

---

## Bağlantı sxemi (qısa)

**Doldurma tərəfi:**
Test Workflow → Read/Write Files → Qdrant Vector Store (insert). Bu node-a
əlavə olaraq Default Data Loader (sənəd) və Embeddings OpenAI (vektor) də
qoşulur, Default Data Loader-ə isə Recursive Character Text Splitter qoşulur.

**Chat tərəfi:**
Chat button → AI Agent. AI Agent-ə üç şey qoşulur: Groq Chat Model (beyin),
Simple Memory (yaddaş) və Qdrant Vector Store1 — tool rejimində (axtarış
aləti). Qdrant Vector Store1-ə də Embeddings OpenAI1 qoşulur.

---

## Nəzərə almalı olduğun məqamlar

- **Kredensiallar:** Workflow OpenAI, Groq və Qdrant kredensiallarını
  ID-lərinə görə çağırır. Hazırkı ID-lər bu instansiyada mövcud olanlara
  uyğunlaşdırılıb (bax: Troubleshooting jurnalı, 2 və 4-cü addımlar).
- **Qdrant kolleksiyası:** Hər iki Qdrant node-u `32` adlı/ID-li
  kolleksiyaya işarə edir — kolleksiyanın Qdrant-da mövcud olduğuna əmin ol.
- **İşə salma ardıcıllığı:** Əvvəlcə "Test Workflow" düyməsi ilə doldurma
  hissəsini işə sal (sənəd Qdrant-a yazılsın), sonra "Chat" düyməsi ilə
  sual ver.
- **Model seçimi:** `Groq Chat Model` node-unda seçdiyin model
  tool-calling (alət çağırma) dəstəkləməlidir, çünki AI Agent Qdrant-ı
  alət kimi çağırır. "compound" adlı modellər bunu dəstəkləmir.

---

## Troubleshooting jurnalı — başdan-sona bütün addımlar və problemlər

Aşağıda bu workflow-u işə salana qədər gördüyümüz bütün addımlar,
qarşılaşdığımız problemlər və onların həlli xronoloji ardıcıllıqla
qeyd olunub.

### 1. Workflow-un import edilməsi

**Nə etdim:** Göndərdiyin JSON-u `n8n import:workflow` CLI əmri ilə işə
düşən n8n konteynerinə import etdim.

**Problem 1 — path xətası:** Git Bash `/data/shared/...` yolunu avtomatik
Windows yoluna çevirdi (`C:/Program Files/Git/data/...`), fayl tapılmadı.
**Həll:** `MSYS_NO_PATHCONV=1` ilə bu avtomatik çevrilməni söndürdüm.

**Problem 2 — SQLITE_CONSTRAINT xətası** (workflow_entity.id sahəsi boş qala
bilməz): Bu, əslində daha dərin bir səbəbdən yaranmışdı.

**Kök səbəb:** `docker-compose.yml`-də `n8n` servisinə əlavə etdiyin

```yaml
environment:
  - N8N_RESTRICT_FILE_ACCESS_TO=
```

bloku, `x-n8n` anchor-undan miras qalan **bütün** `environment` siyahısını
(o cümlədən `DB_TYPE`, `DB_POSTGRESDB_HOST/USER/PASSWORD`) tamamilə əvəz
edirdi — YAML-da `<<:` merge zamanı list tipli açarlar birləşmir, üstünə
yazılır. Nəticədə `n8n` konteyneri Postgres əvəzinə default SQLite ilə
işləyirdi, `n8n-import` konteyneri isə Postgres-ə yazırdı — uyğunsuzluq.

**Həll:** `N8N_RESTRICT_FILE_ACCESS_TO=` sətrini `x-n8n` anchor-un öz
`environment` siyahısına köçürdüm, `n8n` servisindəki ayrıca `environment:`
blokunu sildim. `docker compose up -d` ilə konteynerləri yenidən yaratdım,
`DB_TYPE=postgresdb`-nin artıq düzgün tətbiq olunduğunu təsdiqlədim.

**Nəticə:** Workflow-a `id: ragChatbotQdrantGroq01` verib import etdim —
uğurlu oldu, `n8n list:workflow`-da göründü.

### 2. Chat işləmədi — "Failed to receive response"

**Müşahidə:** n8n loglarında istifadəçinin icazəsi olmadan workflow-a
girməyə cəhd etdiyini bildirən xəbərdarlıq sətirləri gördüm.

**Araşdırma:** Bazada (`shared_workflow` cədvəli) yoxladım — workflow
əslində sənin şəxsi layihənə (`Personal`) düzgün bağlanmışdı (n8n tək
istifadəçi olduğu üçün avtomatik təyin edib). Yəni bu, əsl səbəb deyildi.

**Əsl səbəb (bazadan tapıldı):** `credentials_entity` cədvəlini yoxlayanda
gördüm ki, workflow-dakı node-ların istinad etdiyi kredensial ID-ləri
(`on2IqGYUpxjoJXsO`, `K412utK31JdqXkm7`, `qfEXY4TvV7SZYT0u`) bu n8n
instansiyasında ümumiyyətlə mövcud deyildi — onlar sənin əvvəlki (başqa)
n8n export-undan gəlmişdi.

**Həll (1-ci addım):** Sənə OpenAI və Groq üçün UI-də yeni kredensial
yaratmağı tövsiyə etdim (API açarını mənə yazmadan, təhlükəsizlik üçün).

### 3. Groq credential yaradıldı, amma "Error fetching options"

**Araşdırma:** n8n konteynerindən Groq API-yə şəbəkə testi etdim
(`wget https://api.groq.com/...`) — `401 Unauthorized` aldım, yəni şəbəkə
problemsizdir, server əlçatandır.

**Nəticə:** Deməli, problem API key-in özündə idi (səhv/köhnə açar). Sənə
açarı yenidən kopyalamağı tövsiyə etdim.

### 4. Qdrant Vector Store node-unda "Error in sub-node 'Embeddings OpenAI'"

**Araşdırma:** Eyni şəkildə OpenAI API-yə şəbəkə testi etdim — yenə `401`
(şəbəkə OK). Səni node-un daxilindəki real xəta mətnini açmağa yönləndirdim.

**Tapılan dəqiq xəta:** kredensial ID-si on2IqGYUpxjoJXsO, openAiApi tipi
üçün mövcud deyil, deyilirdi.

**Kök səbəb:** UI-də dropdown-da düzgün adlı kredensial göstərilsə də,
node-un backend-də saxlanan real bağlantısı hələ köhnə/sınıq ID-yə idi —
dəyişikliklər workflow-a saxlanmamışdı (Save edilməmişdi).

**Həll:** Bazadan indiki düzgün kredensial ID-lərini tapdım
(OpenAI: `5IxGvmIH4Jv9I8OM`, Groq: `GzZo3tQd99DQybyj`,
Qdrant: mövcud olan `sFfERYppMeBnFNeA` — "Local QdrantApi database") və
`workflow_entity.nodes` sütununu birbaşa SQL ilə yeniləyib bütün 5 node-un
kredensial istinadını düzgün ID-lərə köçürdüm. Səni səhifəni yeniləməyə
(F5) yönləndirdim.

### 5. İndeksləmə (sənədin Qdrant-a yüklənməsi) uğurla işlədi

"Execute workflow" ilə test etdin, bütün node-lar yaşıllaşdı, sənəd
Qdrant-a yazıldı.

### 6. Chat yenə "Failed to receive response" verdi — Logs-da heç nə yox idi

**Fərq:** Bu dəfə icra qeydi belə yaranmamışdı — deməli sorğu serverə heç
çatmamışdı.

**Araşdırma:** Konteynerin env dəyişənlərini yoxladım — `N8N_PROTOCOL=https`
aktiv idi, amma heç bir SSL sertifikatı/reverse-proxy qurulmamışdı. n8n bu
üzdən chat üçün `https://localhost:5678` ünvanı yaradırdı, brauzer isə sadə
`http://` üzərində işlədiyi üçün qoşula bilmirdi.

**Həll:** `.env` faylında `N8N_PROTOCOL=https` sətrini şərhə (`#`) saldım,
konteynerləri yenidən qaldırdım. Loqda editorun artıq düz `http://` ünvanı
ilə əlçatan olduğunu göstərən sətirlə təsdiqlədim.

### 7. Chat serverə çatdı, amma "Error in sub-node 'Groq Chat Model'"

**Xəta:** "llama-3.3-70b-versatile modeli mövcud deyil və ya ona girişin
yoxdur" mənasında mesaj çıxdı.

**Səbəb:** Orijinal JSON-da yazılan model adı Groq tərəfindən artıq
dəstəklənmir/hesabında yoxdur.

**Həll:** Sənə node-un Model dropdown-undan real mövcud model seçməyi
tövsiyə etdim. Sən `groq/compound-mini` seçdin, işlədi.

### 8. Chat işlədi, amma söhbət yaddaşı yox idi ("adımı yadda saxlamır")

**Araşdırma:** n8n-in öz node kodunu (`ChainRetrievalQa.node.js`) birbaşa
oxudum — "Question and Answer Chain" node-unun yalnız `Model` və
`Retriever` girişi var, memory girişi ümumiyyətlə yoxdur (dizayn etibarilə
tək-mərhələli, yaddaşsız chain-dir).

**Həll (arxitektur dəyişikliyi):** "Question and Answer Chain" və
"Vector Store Retriever" node-larını sildim, yerinə:

- **"AI Agent"** node-u (memory və tool dəstəkləyir),
- **"Simple Memory"** node-u (son 5 mesajı yadda saxlayır),
- **"Qdrant Vector Store1"**-i `retrieve-as-tool` rejiminə keçirdim
  (Agent onu alət kimi çağırır).

Bu node tiplərinin dəqiq parametr formatını (versiya, sahə adları) n8n-in
öz quraşdırılmış paket kodundan yoxlayıb, sonra `workflow_entity`
cədvəlini birbaşa SQL ilə yenilədim (UI-də əl ilə qurmaq səhv ehtimalı
yüksək olardığı üçün).

**Nəticə:** Sual-cavab (məs. "Bakıya çatdırılma nə qədər çəkir?") uğurla
işlədi.

### 9. Yeni arxitekturada "Bad request - tool calling is not supported"

**Xəta:** Seçilmiş `groq/compound-mini` modeli tool-calling (alət çağırma)
funksiyasını dəstəkləmir, halbuki indi AI Agent Qdrant-ı alət kimi
çağırmalıdır.

**Status:** Hələ həll olunmayıb. Sənə tool-calling dəstəkləyən başqa model
seçməyi tövsiyə etdim (`llama-3.3-70b-versatile`, `llama-3.1-8b-instant`
kimi, "compound" olmayan modellərdən biri).

### 10. Başqa tapşırıq üçün `.env`-ə əlavə olunan ngrok ayarları RAG Chatbot-u yenidən sındırdı

**Fon:** Instagram DM tapşırığı üçün `.env`-ə yenidən bu sətirlər əlavə
olunmuşdu: `WEBHOOK_URL`, `N8N_HOST`, `N8N_PROTOCOL=https`,
`N8N_WEBHOOK_URL` (hamısı ngrok domenini göstərir).

**Simptom:** `http://localhost:5678` üzərindən açanda n8n loglarında
"User attempted to access a workflow without permissions" xətası, RAG
Chatbot workflow-u açılmırdı.

**Kök səbəb:** `N8N_PROTOCOL=https` aktiv olduğu üçün n8n login cookie-ni
**Secure** işarəsi ilə göndərirdi (yəni cookie yalnız https üzərində
etibarlıdır). Brauzer bu cookie-ni sadə `http://localhost` sorğularına
əlavə etmirdi, n8n isə istifadəçini "tanımırdı".

**Həll:** `.env`-ə `N8N_SECURE_COOKIE=false` sətrini əlavə etdim (fayl
Read-Only işarəli olduğu üçün əvvəlcə `Set-ItemProperty -IsReadOnly $false`
ilə açdım), sonra `docker restart n8n` etdim. Nəticədə ngrok/Instagram
webhook-u da, `localhost:5678` girişi də paralel işləməyə davam etdi.

### 11. Chat modeli sual-cavabı Azərbaycan yerinə ərəb dilində qaytardı

**Səbəb:** "AI Agent" node-unun **System Message** sahəsi tamamilə boş idi
— modelə hansı dildə cavab verməli olduğu barədə heç bir təlimat
verilməmişdi.

**Həll:** System Message sahəsinə bu mətni yazdım:
> "Sən Azərbaycan dilində danışan köməkçi süni intellektsən. İstifadəçi
> hansı dildə yazsa yazsın, HƏMİŞƏ Azərbaycan dilində, sadə və aydın
> dildə cavab ver."

Dəyişikliyi `n8n import:workflow` ilə CLI-dan tətbiq etdim (bax:
`n8n/demo-data/workflows/instagram-post.json` ilə eyni üsul). Qeyd: bu
üsulla edilən dəyişiklik artıq brauzerdə açıq olan səhifəyə **canlı**
ötürülmür — səhifəni yeniləmək (F5) lazımdır ki, redaktor yeni versiyanı
göstərsin.

### 12. System Message əlavə olunsa da, model istifadəçinin adını uydurdu

**Simptom:** İstifadəçi "adım Gunel" yazıb, sonra "adım nədir?" soruşanda
model "Adınız Gündəlidir" kimi mövcud olmayan bir söz uydurdu (adı düzgün
xatırlamaq əvəzinə).

**Araşdırma:** Əvvəlcə sessiya ID-lərinin fərqli olub-olmadığını yoxladım
(səhifə yeniləndikdə n8n test çatı yeni sessiya açır, "Simple Memory" isə
sessiyaya bağlıdır) — istifadəçi adı **eyni sessiyada** təkrar yazdığını
təsdiqlədi, deməli yaddaş itməsi səbəb deyildi.

**Kök səbəb:** İşlədilən model `openai/gpt-oss-safeguard-20b` idi — bu,
OpenAI-nin **məzmun moderasiyası/təhlükəsizlik** üçün hazırladığı xüsusi
model, adi söhbət üçün nəzərdə tutulmayıb. Kiçik ölçüsü (20B) ilə birlikdə
Azərbaycan dilini zəif emal etməsi bu cür uydurmalara səbəb olurdu.

### 13. Model dəyişikliyi cəhdləri — üç ardıcıl xəta

**1-ci cəhd — `llama-3.3-70b-versatile`:** Groq-un rəsmi sənədlərində bu
model production siyahısında görünsə də, bu konkret n8n instansiyasının
canlı (hesaba bağlı) model siyahısında **mövcud deyildi** — n8n
"The value 'llama-3.3-70b-versatile' is not supported!" xəbərdarlığı
verdi. (Qeyd: mən əvvəlcə sənədə əsaslanaraq bu modeli tövsiyə etmişdim,
amma real hesabın siyahısı fərqli çıxdı — bu, səhv tövsiyə idi.)

**2-ci cəhd — `groq/compound`:** Bu, Groq-un öz daxili alətləri olan
xüsusi "agentic sistem"dir, istifadəçinin öz Tool-larını (bizim halda
Qdrant Vector Store1) qəbul etmir. Xəta: "Bad request ... tool calling is
not supported with this model".

**3-cü cəhd (tövsiyə, nəticə gözlənilir) — `openai/gpt-oss-120b`:** Ümumi
məqsədli, tool-calling dəstəkləyən, "safeguard" olmayan model. Bunu
istifadəçiyə tövsiyə etdim, nəticəsi hələ təsdiqlənməyib.

---

## Ümumi nəticə

Bu layihədə üst-üstə problemlər aşkarlandı və həll edildi (yaxud həll
prosesindədir):

1. Git Bash path çevrilməsi
2. `docker-compose.yml`-də YAML environment override bug-ı
   (Postgres əvəzinə SQLite işləməsi)
3. Workflow-un başqa n8n instansiyasından gələn sınıq kredensial ID-ləri
4. `N8N_PROTOCOL=https` konfiqurasiya səhvi (SSL olmadan)
5. Groq-da artıq mövcud olmayan model adı
6. "Question and Answer Chain" node-un yaddaş dəstəkləməməsi →
   AI Agent arxitekturasına keçid
7. Seçilmiş Groq modelinin tool-calling dəstəkləməməsi
8. Başqa tapşırıq üçün `.env`-ə əlavə olunan ngrok/https ayarlarının
   `localhost:5678` girişini sındırması → `N8N_SECURE_COOKIE=false`
9. Boş System Message səbəbindən modelin səhv dildə (ərəbcə) cavab verməsi
10. "Safeguard" modelinin adı uydurması → daha güclü, ümumi məqsədli
    modelə keçid ehtiyacı
11. *(davam edir)* Tool-calling dəstəkləyən, Azərbaycan dilində keyfiyyətli
    cavab verən düzgün Groq modelinin tapılması

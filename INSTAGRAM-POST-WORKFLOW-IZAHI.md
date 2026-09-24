# Instagram Post Workflow — İzahı

Bu sənəd "Instagram & LinkedIn Share" dərsi üçün gördüyüm işləri və workflow-un
necə işlədiyini addım-addım izah edir.

## Nə edildi

1. `n8n/demo-data/workflows/instagram-post.json` faylı yaradıldı — dərsdəki
   hazır JSON şablonu əsasında (bölmə 6.4).
2. Bu fayl işləyən **n8n** konteynerinə köçürülüb (`docker cp`), sonra
   `n8n import:workflow` əmri ilə import edildi.
3. Workflow indi `http://localhost:5678` panelində görünür, adı:
   **"Instagram Post (HTTP Request)"**, id: `instagramPostHttpRequest01`.

Workflow **passiv (inactive)** vəziyyətdə saxlanıldı — çünki içində hələ real
token və IG User ID yoxdur, yanlışlıqla işə düşüb boş post atmasın deyə.

## Workflow-un strukturu

```
Form Trigger  →  IG: Container Yarat  →  5 saniyə gözlə  →  IG: Postu Yayımla
```

### 1. `Form Trigger`
n8n içində sadə bir form açır, iki sahə istəyir: `image_url` (şəkil linki) və
`caption` (yazı). Bu, dərsdə deyilən "n8n-ə lazım olan giriş nöqtəsi"dir.

### 2. `IG: Container Yarat`
`HTTP Request` node-udur (Facebook Graph API node-u **yox** — dərsin 6.1-ci
bölməsində izah olunduğu kimi, həmin hazır node n8n 2.x-də query
parametrlərini düzgün göndərmir).

- Metod: `POST`
- URL: `https://graph.facebook.com/v21.0/IG_USER_ID/media`
- Autentifikasiya: `Header Auth` tipli credential (`Authorization: Bearer ...`)
- Query parametrləri: `image_url` və `caption`, **Expression** rejimində
  (`={{ $json.image_url }}`)

Bu addımda Meta şəkli öz serverinə çəkir və bir "container" yaradır, cavabda
`id` (creation_id) qaytarır.

### 3. `5 saniyə gözlə`
`Wait` node-u. Şəklin emalı üçün Meta-ya vaxt verir — dərhal `media_publish`-ə
keçsə, `Media ID is not available` xətası çıxır.

### 4. `IG: Postu Yayımla`
İkinci `HTTP Request` node-u:

- URL: `https://graph.facebook.com/v21.0/IG_USER_ID/media_publish`
- `creation_id` sahəsi `{{ $json.id }}` əvəzinə birbaşa
  `{{ $('IG: Container Yarat').item.json.id }}` ilə göndərilir.

Bu, dərsdəki 7-ci bölmədə tövsiyə olunan detaldır: birbaşa node adına
istinad etsən, gələcəkdə flow-a yeni node əlavə etdikdə (məsələn status
yoxlama addımı) `$json` axını dəyişsə belə, bu sətir sınmır.

Uğurlu cavabda Instagram-dakı real post ID-si qayıdır — bu, postun canlı
yayımlandığını göstərir.

## Sənin görməli olduğun işlər (mən sənin adından edə bilmədiyim hissə)

Bunlar Facebook/Instagram hesabına giriş və şəxsi təsdiq tələb etdiyi üçün
mən avtomatik edə bilmirəm — brauzerdə özün görməlisən:

1. **Instagram hesabını Professional (Business/Creator) et** və Facebook
   Page-ə bağla.
2. **Meta App yarat** (developers.facebook.com), Instagram məhsulunu əlavə
   et, "API setup with Facebook login" seç.
3. **System User yarat**, ona app-ı təyin et, `Never` bitmə tarixli token
   generasiya et (6 permission ilə: `instagram_basic`,
   `instagram_content_publish`, `pages_show_list`,
   `pages_read_engagement`, `pages_manage_posts`, `business_management`).
4. `curl .../me/accounts?fields=...instagram_business_account...` ilə real
   **IG User ID**-ni tap (`17841...` formatında).
5. n8n panelində **Credentials → New → Header Auth** yarat:
   - Name: `Authorization`
   - Value: `Bearer <TOKEN>`
   sonra hər iki `HTTP Request` node-unda bu credential-ı seç (hazırkı
   fayldakı `META_HEADER_CRED` sadəcə boş yer tutandır).
6. Hər iki node-un URL-ində `IG_USER_ID` sözünü öz real ID-nlə əvəz et.
7. Workflow-u **Active** et (və ya "Test workflow" ilə əl ilə işə sal),
   formu doldur, nəticəni öz Instagram səhifəndə yoxla.

Bunları etdikdən sonra mənə de — istəsən addımları birlikdə yoxlayaq, ya da
xəta çıxarsa (dərsin 8-ci bölməsindəki xəta cədvəlinə uyğun) birlikdə həll
edərik.

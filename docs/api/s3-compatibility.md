# Hafiz S3 API Uyumluluk Matrisi

Bu dokuman, Hafiz nesne depolama sisteminin Amazon S3 API'si ile uyumluluk durumunu detayli olarak belgelemektedir.

**Son Guncelleme:** 2026-02-09

---

## 1. Genel Bakis

Hafiz, Rust dilinde yazilmis kurumsal duzeyde bir S3 uyumlu nesne depolama sistemidir. Asagidaki temel ozellikleri destekler:

- **Tam S3 API uyumlulugu:** Bucket ve nesne operasyonlari, multipart upload, versiyonlama
- **Metadata altyapisi:** SQLite (varsayilan, tek dugum) ve PostgreSQL (kumeleme icin onerilen)
- **Depolama motoru:** Yerel dosya sistemi tabanli depolama
- **Kimlik dogrulama:** AWS Signature V4, LDAP/Active Directory entegrasyonu, presigned URL'ler
- **Sifreleme:** Sunucu tarafli sifreleme (SSE-S3, SSE-C) - AES-256-GCM
- **Versiyonlama:** Tam nesne versiyonlama ve silme isaretcileri (delete marker) destegi
- **Object Lock:** WORM (Write Once Read Many) uyumlulugu, retention ve legal hold
- **Yasam dongusu yonetimi:** Lifecycle kurallari ile otomatik nesne yonetimi
- **Kumeleme:** Cok dugumlu replikasyon ve dugum kesfetme (opsiyonel)
- **Etiketleme:** Bucket ve nesne duzeyinde etiketleme (tagging)
- **CORS:** Cross-Origin Resource Sharing yapilandirmasi
- **Bildirim:** Webhook, Queue ve Topic tabanli olay bildirimleri
- **Erisim kontrolu:** Bucket policy (JSON), ACL (XML), canned ACL destegi

### Durum Aciklamalari

| Durum | Anlami |
|-------|--------|
| **Uygulanmis** | Tam olarak uygulanmis ve uretimde kullanilabilir |
| **Kismi** | Temel islevsellik mevcut ancak bazi alt ozellikler eksik |
| **Planlanmis** | Henuz uygulanmamis, gelecek surumler icin planlanmis |
| **Uygulanmayacak** | Hafiz mimarisine uygun olmadigi icin uygulanmasi planlanmiyor |

---

## 2. Bucket (Kova) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `CreateBucket` (PUT /) | **Uygulanmis** | Bucket adi dogrulama, metadata + depolama dizini olusturma. Bucket adi kurallari AWS S3 ile uyumlu. |
| `DeleteBucket` (DELETE /) | **Uygulanmis** | Bosaltilmis bucket silme. Metadata ve depolama dizini birlikte temizlenir. |
| `HeadBucket` (HEAD /) | **Uygulanmis** | Bucket varlik kontrolu. 200 OK veya 404 NoSuchBucket donduruyor. |
| `ListBuckets` (GET /) | **Uygulanmis** | Sahibe gore bucket listeleme. XML yaniti S3 formatiyla uyumlu. |
| `GetBucketLocation` | **Planlanmis** | Bolge bilgisi dondurmesi henuz eklenmedi. |
| `ListObjectsV1` (GET /?prefix=&marker=) | **Uygulanmis** | Prefix, delimiter, max-keys, marker parametreleri destekleniyor. |
| `ListObjectsV2` (GET /?list-type=2) | **Uygulanmis** | continuation-token, prefix, delimiter, max-keys parametreleri destekleniyor. `list-type=2` sorgu parametresi ile etkinlesir. |

---

## 3. Nesne (Object) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetObject` (GET /{key}) | **Uygulanmis** | Range istekleri (byte-range), versiyon destegi (`versionId`), SSE baslik bilgileri, kullanici metadata'si (`x-amz-meta-*`), delete marker tespiti destekleniyor. |
| `PutObject` (PUT /{key}) | **Uygulanmis** | Content-Type otomatik algulama (mime_guess), SSE-S3 ve SSE-C baslik destegi, kullanici metadata'si, versiyonlama destegi. Kumeleme modunda asenkron replikasyon olay kuyrugu. |
| `DeleteObject` (DELETE /{key}) | **Uygulanmis** | Basit silme ve versiyonlu silme. Versiyonlama aciksa delete marker olusturma, belirli bir versiyonu silme (`versionId`). Object Lock kontrolu (retention + legal hold). Kumeleme modunda replikasyon destegi. |
| `DeleteObjects` (POST /?delete) | **Uygulanmis** | Toplu silme (multi-object delete). Quiet modu destegi. XML istek/yanit formati S3 uyumlu. |
| `HeadObject` (HEAD /{key}) | **Uygulanmis** | Content-Type, Content-Length, ETag, Last-Modified baslik bilgileri donduruluyor. Kullanici metadata'si (`x-amz-meta-*`) yanit basliklarinda destekleniyor. |
| `CopyObject` (PUT /{key} + x-amz-copy-source) | **Uygulanmis** | Farkli bucket'lar arasi kopyalama, `x-amz-metadata-directive` (COPY/REPLACE) destegi, URL decode edilmis kaynak anahtar, hedef bucket varlik dogrulama. |
| `GetObjectVersion` (GET /{key}?versionId=) | **Uygulanmis** | Belirli bir nesne versiyonunu getirme. Delete marker tespiti, Range destegi, SSE baslik bilgileri. |
| `DeleteObjectVersion` (DELETE /{key}?versionId=) | **Uygulanmis** | Belirli bir versiyonu silme. Object Lock kontrolu entegrasyonu. |
| `PostObject` (POST /) | **Planlanmis** | Form tabanli yukleme henuz uygulanmadi. |
| `SelectObjectContent` (POST /{key}?select&select-type=2) | **Uygulanmis** | SQL tabanli nesne icerigi sorgulama. CSV (header, delimiter, quote), JSON (DOCUMENT, LINES) girdi destegi. SELECT/WHERE/LIMIT SQL alt kumesi. zstd sikistirilmis nesnelerde seffaf calisir. |
| `RestoreObject` | **Uygulanmayacak** | Glacier-tarz arsiv depolama sinifi desteklenmiyor. |

---

## 4. Multipart Upload (Coklu Parca Yukleme) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `CreateMultipartUpload` (POST /{key}?uploads) | **Uygulanmis** | Benzersiz upload ID olusturma, content-type ve kullanici metadata'si destegi. |
| `UploadPart` (PUT /{key}?uploadId=&partNumber=) | **Uygulanmis** | Parca numarasi dogrulama (1-10000), parca verisi depolama, ETag hesaplama. Parca depolama formati: `{key}/.parts/{uploadId}/{partNumber}`. |
| `CompleteMultipartUpload` (POST /{key}?uploadId=) | **Uygulanmis** | Parca birlesimi, nihai ETag hesaplama (MD5 birlestirme formati), parca temizligi, nesne metadata olusturma. Kumeleme modunda asenkron replikasyon olay kuyrugu. |
| `AbortMultipartUpload` (DELETE /{key}?uploadId=) | **Uygulanmis** | Yukleme iptali, parca verilerinin temizlenmesi, upload kaydinin silinmesi. |
| `ListParts` (GET /{key}?uploadId=) | **Uygulanmis** | Parca listeleme, max-parts ve part-number-marker ile sayfalama destegi. |
| `ListMultipartUploads` (GET /?uploads) | **Uygulanmis** | Devam eden yuklemeleri listeleme. Prefix, key-marker, upload-id-marker, max-uploads parametreleri destekleniyor. |
| `UploadPartCopy` | **Planlanmis** | Kopyalama ile parca yukleme henuz uygulanmadi. |

---

## 5. Versiyonlama (Versioning) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketVersioning` (GET /?versioning) | **Uygulanmis** | Bucket versiyonlama durumunu sorgulama (Enabled/Suspended/bos). |
| `PutBucketVersioning` (PUT /?versioning) | **Uygulanmis** | Versiyonlama etkinlestirme/askiya alma. Object Lock etkinken versiyonlamanin devre disi birakilmasi engellenir. XML istek govdesi dogrulama. |
| `ListObjectVersions` (GET /?versions) | **Uygulanmis** | Nesne versiyonlarini ve delete marker'lari listeleme. Prefix, delimiter, max-keys, key-marker, version-id-marker parametreleri destekleniyor. |
| `CreateDeleteMarker` | **Uygulanmis** | Versiyonlu bucket'ta silme islemi otomatik olarak delete marker olusturur. |
| `MFA Delete` | **Planlanmis** | MFA ile korunmus versiyonlama henuz uygulanmadi. |

---

## 6. Yasam Dongusu (Lifecycle) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketLifecycleConfiguration` (GET /?lifecycle) | **Uygulanmis** | Lifecycle kural yapilandirmasini sorgulama. XML yaniti S3 uyumlu. |
| `PutBucketLifecycleConfiguration` (PUT /?lifecycle) | **Uygulanmis** | Lifecycle kurallari tanimlama. Kural dogrulama, XML ayristirma. |
| `DeleteBucketLifecycle` (DELETE /?lifecycle) | **Uygulanmis** | Lifecycle yapilandirmasini silme. |
| Lifecycle Kurallari - Expiration | **Uygulanmis** | Nesne suresi dolma kurali. `get_objects_for_lifecycle` ve `get_buckets_with_lifecycle` ile otomatik isleme. |
| Lifecycle Kurallari - NoncurrentVersionExpiration | **Uygulanmis** | Gecersiz versiyonlarin suresi dolma kurali. |
| Lifecycle Kurallari - Transition | **Planlanmis** | Depolama sinifi gecisi henuz desteklenmiyor (tek depolama sinifi: STANDARD). |
| Lifecycle Kurallari - AbortIncompleteMultipartUpload | **Planlanmis** | Tamamlanmamis multipart upload temizligi henuz otomatik degil. |

---

## 7. Etiketleme (Tagging) Operasyonlari

### Bucket Etiketleme

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketTagging` (GET /?tagging) | **Uygulanmis** | Bucket etiketlerini sorgulama. TagSet XML formati. |
| `PutBucketTagging` (PUT /?tagging) | **Uygulanmis** | Bucket etiketlerini ayarlama. XML ayristirma ve dogrulama. |
| `DeleteBucketTagging` (DELETE /?tagging) | **Uygulanmis** | Bucket etiketlerini silme. |

### Nesne Etiketleme

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetObjectTagging` (GET /{key}?tagging) | **Uygulanmis** | Nesne etiketlerini sorgulama. Versiyon destegi (`versionId`). |
| `PutObjectTagging` (PUT /{key}?tagging) | **Uygulanmis** | Nesne etiketlerini ayarlama. Versiyon destegi. XML ayristirma. |
| `DeleteObjectTagging` (DELETE /{key}?tagging) | **Uygulanmis** | Nesne etiketlerini silme. Versiyon destegi. |

---

## 8. CORS (Cross-Origin Resource Sharing) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketCors` (GET /?cors) | **Uygulanmis** | CORS yapilandirmasini sorgulama. Yapilandirma yoksa `NoSuchCORSConfiguration` hatasi. |
| `PutBucketCors` (PUT /?cors) | **Uygulanmis** | CORS kurallarini ayarlama. XML ayristirma, dogrulama (CorsConfiguration), temiz XML'e serializasyon. AllowedOrigin, AllowedMethod, AllowedHeader, ExposeHeader, MaxAgeSeconds destegi. |
| `DeleteBucketCors` (DELETE /?cors) | **Uygulanmis** | CORS yapilandirmasini silme. |
| `OPTIONS Preflight` (OPTIONS /{bucket}/{key}) | **Uygulanmis** | Preflight isteklerini isleme. Origin, Access-Control-Request-Method, Access-Control-Request-Headers basliklarini kontrol etme. Eslesen CORS kurali bulma, izin verilen/reddedilen yanit basliklarini ayarlama. |
| CORS Yanit Basliklarini Ekleme | **Uygulanmis** | Gercek (preflight olmayan) isteklere CORS basliklarini ekleme. `add_cors_headers_to_response` middleware fonksiyonu. |
| Wildcard Origin Eslestirme | **Uygulanmis** | `*` wildcard origin destegi. |

---

## 9. ACL (Access Control List) Operasyonlari

### Bucket ACL

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketAcl` (GET /?acl) | **Uygulanmis** | Bucket ACL'ini sorgulama. Yapilandirma yoksa varsayilan `private` ACL dondurulur. |
| `PutBucketAcl` (PUT /?acl) | **Uygulanmis** | Uc farkli yontemle ACL ayarlama destegi: (1) `x-amz-acl` basligi ile canned ACL, (2) XML govde ile tam ACL belgesi, (3) `x-amz-grant-*` basliklarla grant tabanli ACL. |

### Nesne ACL

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetObjectAcl` (GET /{key}?acl) | **Uygulanmis** | Nesne ACL'ini sorgulama. Versiyon destegi. Yapilandirma yoksa varsayilan `private` ACL dondurulur. |
| `PutObjectAcl` (PUT /{key}?acl) | **Uygulanmis** | Bucket ACL ile ayni uc yontem destekleniyor. Versiyon destegi. |

### Desteklenen Canned ACL Degerleri

| Canned ACL | Durum |
|------------|-------|
| `private` | **Uygulanmis** |
| `public-read` | **Uygulanmis** |
| `public-read-write` | **Uygulanmis** |
| `authenticated-read` | **Uygulanmis** |
| `bucket-owner-read` | **Uygulanmis** |
| `bucket-owner-full-control` | **Uygulanmis** |

### Desteklenen Grant Basliklarri

| Baslik | Durum |
|--------|-------|
| `x-amz-grant-read` | **Uygulanmis** |
| `x-amz-grant-write` | **Uygulanmis** |
| `x-amz-grant-read-acp` | **Uygulanmis** |
| `x-amz-grant-write-acp` | **Uygulanmis** |
| `x-amz-grant-full-control` | **Uygulanmis** |

---

## 10. Bucket Policy Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketPolicy` (GET /?policy) | **Uygulanmis** | Bucket politikasini JSON olarak sorgulama. Politika yoksa `NoSuchBucketPolicy` hatasi. |
| `PutBucketPolicy` (PUT /?policy) | **Uygulanmis** | JSON policy belgesi dogrulama (PolicyDocument yapisina uygunluk), en az bir statement zorunlulugu. Metadata'da JSON olarak saklanir. |
| `DeleteBucketPolicy` (DELETE /?policy) | **Uygulanmis** | Bucket politikasini silme. |
| `GetBucketPolicyStatus` | **Planlanmis** | Politika durumu sorgulama henuz uygulanmadi. |

---

## 11. Object Lock / Retention / Legal Hold Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetObjectLockConfiguration` (GET /?object-lock) | **Uygulanmis** | Object Lock yapilandirmasini sorgulama. Yapilandirma yoksa `ObjectLockConfigurationNotFoundError`. |
| `PutObjectLockConfiguration` (PUT /?object-lock) | **Uygulanmis** | Object Lock etkinlestirme. XML ayristirma ve dogrulama. Etkinlestirildikten sonra guncelleme destegi (varsayilan retention degisikligi). |
| `GetObjectRetention` (GET /{key}?retention) | **Uygulanmis** | Nesne retention yapilandirmasini sorgulama. Versiyon destegi. Object Lock etkin kontrolu zorunlu. |
| `PutObjectRetention` (PUT /{key}?retention) | **Uygulanmis** | Nesne retention ayarlama. `x-amz-bypass-governance-retention` basligi ile Governance modu atlama. Mevcut retention kontrolu (degistirilip degistirilemeyecegi). |
| `GetObjectLegalHold` (GET /{key}?legal-hold) | **Uygulanmis** | Legal hold durumunu sorgulama. Legal hold yoksa OFF durumu dondurulur. Versiyon destegi. |
| `PutObjectLegalHold` (PUT /{key}?legal-hold) | **Uygulanmis** | Legal hold etkinlestirme/devre disi birakma (ON/OFF). XML ayristirma. Versiyon destegi. |

### Object Lock Koruma Kontrolleri

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| Silme oncesi Object Lock kontrolu | **Uygulanmis** | `can_delete_object` fonksiyonu legal hold ve retention kontrolu yapar. |
| Governance modu atlama | **Uygulanmis** | `x-amz-bypass-governance-retention: true` basligi ile destekleniyor. |
| Compliance modu | **Uygulanmis** | Compliance modundaki nesneler suresi dolmadan degistirilemez. |
| Varsayilan retention kurali | **Uygulanmis** | Bucket duzeyli varsayilan retention yapilandirmasi. |

---

## 12. Bildirim (Notification) Operasyonlari

| API Adi | Durum | Notlar |
|---------|-------|--------|
| `GetBucketNotificationConfiguration` (GET /?notification) | **Uygulanmis** | Bildirim yapilandirmasini sorgulama. JSON'dan XML'e donusturme. Yapilandirma yoksa bos yapilandirma dondurulur. |
| `PutBucketNotificationConfiguration` (PUT /?notification) | **Uygulanmis** | Bildirim yapilandirmasi ayarlama. XML ayristirma, dogrulama, JSON olarak depolama. |

### Desteklenen Bildirim Hedefleri

| Hedef Turu | Durum | Notlar |
|------------|-------|--------|
| `WebhookConfiguration` | **Uygulanmis** | HTTP/HTTPS webhook URL'lerine bildirim. URL dogrulama, filtre kurallari, auth token destegi. Hafiz'e ozel eklenti. |
| `QueueConfiguration` | **Uygulanmis** | ARN tabanli kuyruk bildirimi yapilandirmasi. ARN dogrulama. |
| `TopicConfiguration` | **Uygulanmis** | ARN tabanli konu (topic) bildirimi yapilandirmasi. ARN dogrulama. |
| `LambdaFunctionConfiguration` | **Planlanmis** | Lambda fonksiyon bildirimi henuz desteklenmiyor. |

### Desteklenen Olay Turleri

| Olay | Durum |
|------|-------|
| `s3:ObjectCreated:*` | **Uygulanmis** |
| `s3:ObjectCreated:Put` | **Uygulanmis** |
| `s3:ObjectCreated:Post` | **Uygulanmis** |
| `s3:ObjectCreated:Copy` | **Uygulanmis** |
| `s3:ObjectCreated:CompleteMultipartUpload` | **Uygulanmis** |
| `s3:ObjectRemoved:*` | **Uygulanmis** |
| `s3:ObjectRemoved:Delete` | **Uygulanmis** |
| `s3:ObjectRemoved:DeleteMarkerCreated` | **Uygulanmis** |
| `s3:ObjectTagging:*` | **Uygulanmis** |
| `s3:ObjectTagging:Put` | **Uygulanmis** |
| `s3:ObjectTagging:Delete` | **Uygulanmis** |
| `s3:ObjectAcl:Put` | **Uygulanmis** |
| `s3:ObjectRestore:*` | **Planlanmis** |

---

## 13. Presigned URL'ler

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| Presigned URL olusturma | **Uygulanmis** | `generate_presigned_url` fonksiyonu. GET ve PUT metodlari destekleniyor. |
| Presigned URL dogrulama | **Uygulanmis** | `verify_presigned_url` fonksiyonu. Imza hesaplama ve zaman asimi kontrolu. |
| Presigned istek tespiti | **Uygulanmis** | `is_presigned_request` fonksiyonu. X-Amz-Algorithm ve X-Amz-Signature parametrelerini kontrol eder. |
| Access key cikarma | **Uygulanmis** | `extract_access_key_from_presigned` fonksiyonu. X-Amz-Credential parametresinden cikarim. |
| GET presigned URL | **Uygulanmis** | Nesne indirme icin presigned URL. |
| PUT presigned URL | **Uygulanmis** | Nesne yukleme icin presigned URL. Content-Type ve Content-MD5 baslik destegi. |
| Suresi dolmus URL kontrolu | **Uygulanmis** | `X-Amz-Expires` parametresi ile zaman asimi. `ExpiredPresignedRequest` hatasi. |
| VersionId destegi | **Uygulanmis** | Presigned URL'lerde versiyon belirtme destegi. |
| DELETE presigned URL | **Planlanmis** | Henuz uygulanmadi. |
| Multipart presigned URL | **Planlanmis** | Multipart upload icin presigned URL henuz uygulanmadi. |

---

## 14. Sunucu Tarafli Sifreleme (Server-Side Encryption)

### SSE-S3 (Sunucu Yonetimli Anahtarlar)

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| `x-amz-server-side-encryption: AES256` basligi | **Uygulanmis** | PutObject isteginde SSE-S3 etkinlestirme. AES-256-GCM sifreleme altyapisi (hafiz-crypto crate). |
| `x-amz-server-side-encryption: aws:kms` basligi | **Kismi** | Baslik kabul ediliyor, ancak gercek KMS entegrasyonu yok. SSE-S3 olarak isleniyor. |
| SSE yanit basliklarri | **Uygulanmis** | GetObject ve PutObject yanitlarinda `x-amz-server-side-encryption` basligi. |
| Varsayilan bucket sifreleme | **Planlanmis** | Bucket duzeyi varsayilan sifreleme yapilandirmasi henuz uygulanmadi. |

### SSE-C (Musteri Tarafindan Saglanan Anahtarlar)

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| `x-amz-server-side-encryption-customer-key` basligi | **Uygulanmis** | Base64 kodlanmis musteri anahtari. |
| `x-amz-server-side-encryption-customer-key-md5` basligi | **Uygulanmis** | Anahtar MD5 dogrulama. Yanit basliginda dondurulur. |
| `x-amz-server-side-encryption-customer-algorithm` basligi | **Kismi** | AES256 destekleniyor. |

### SSE-KMS (AWS KMS Entegrasyonu)

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| `x-amz-server-side-encryption: aws:kms` | **Planlanmis** | Gercek KMS entegrasyonu planlanmis. Simdilik SSE-S3 olarak isleniyor. |
| `x-amz-server-side-encryption-aws-kms-key-id` | **Planlanmis** | KMS anahtar ID belirtme destegi planlanmis. |
| Bucket Key | **Planlanmis** | S3 Bucket Key optimizasyonu planlanmis. |

---

## 15. Kimlik Dogrulama (Authentication)

### AWS Signature V4

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| Authorization basligi ayristirma | **Uygulanmis** | `AWS4-HMAC-SHA256 Credential=.../date/region/s3/aws4_request, SignedHeaders=..., Signature=...` formati. |
| Imza hesaplama ve dogrulama | **Uygulanmis** | Kanonik istek olusturma, string-to-sign hesaplama, HMAC-SHA256 imza dogrulama. `verify_signature_v4` fonksiyonu. |
| Credential ayrıstirma | **Uygulanmis** | Access key, tarih, bolge, servis bilgileri cikarilir. |
| Signed headers dogrulama | **Uygulanmis** | Imzalanmis basliklarla kanonik baslik eslestirme. |
| Chunked upload (aws-chunked) | **Planlanmis** | Chunked transfer encoding ile imza destegi henuz uygulanmadi. |
| Unsigned payload | **Uygulanmis** | `UNSIGNED-PAYLOAD` presigned URL'ler icin destekleniyor. |

### LDAP / Active Directory

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| LDAP kimlik dogrulama | **Uygulanmis** | `LdapAuthProvider` ve `LdapClient` ile LDAP sunucu baglantisi. |
| Active Directory destegi | **Uygulanmis** | Microsoft AD uyumlulugu. |
| OpenLDAP destegi | **Uygulanmis** | OpenLDAP ve 389 Directory Server destegi. |
| Kullanici kimlikleme | **Uygulanmis** | LDAP bind ile kullanici dogrulama. |
| Grup tabanli politika eslestirme | **Uygulanmis** | LDAP gruplarina gore bucket erisim politikasi. |
| Kullanici onbellekleme | **Uygulanmis** | Kimlik dogrulanmis kullanicilarin onbellege alinmasi. |
| TLS/STARTTLS destegi | **Uygulanmis** | Guvenli LDAP baglantisi. |
| Ozellestirilebilir ozellik eslestirme | **Uygulanmis** | `AttributeMappings` ile LDAP ozelliklerini Hafiz kullanici alanlarına eslestirme. |

### Erisim Anahtari Yonetimi

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| Access Key / Secret Key cifti olusturma | **Uygulanmis** | AWS uyumlu format (AKIA + 16 karakter access key, 40 karakter secret key). |
| Kimlik bilgisi listeleme | **Uygulanmis** | `list_credentials` API. |
| Kimlik bilgisi guncelleme | **Uygulanmis** | `update_credentials` API. |
| Kimlik bilgisi silme | **Uygulanmis** | `delete_credentials` API. |
| Temporary credentials (STS) | **Planlanmis** | Gecici erisim anahtarlari henuz desteklenmiyor. |

---

## 16. Ek Ozellikler

### Kumeleme (Cluster) Operasyonlari

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| Cok dugumlu replikasyon | **Uygulanmis** | `ClusterManager`, `Replicator` ile asenkron nesne replikasyonu. Kural tanimlanmadan tum saglıklı node'lara otomatik replikasyon. |
| Dugum kesfetme | **Uygulanmis** | `DiscoveryService` ile seed tabanli dugum kesfetme. Cluster HTTP route'lari (`/cluster/join`, `/cluster/heartbeat`, `/cluster/message`, `/cluster/ping`). |
| Saglik kontrolleri | **Uygulanmis** | Dugumler arasi saglik kontrol mekanizmasi. |
| Replikasyon olay kuyrugu | **Uygulanmis** | PutObject, DeleteObject ve CompleteMultipartUpload islemlerinde otomatik replikasyon olaylari. Dongu onleme (`x-hafiz-replication-status: REPLICA`). |
| Replikasyon kurallari | **Uygulanmis** | Bucket bazli replikasyon kurali tanimlama. SQLite ve PostgreSQL backend destegi. |
| PostgreSQL cluster tablolari | **Uygulanmis** | `cluster_nodes` ve `replication_rules` tablolari PostgreSQL backend'de tam CRUD destegi. |
| Replikasyon auth bypass | **Uygulanmis** | Cluster ici replikasyon istekleri `x-hafiz-replication-status: REPLICA` basligi ile kimlik dogrulamayi atlar. |
| Metadata replikasyonu | **Uygulanmis** | Paylasimli PostgreSQL icin REPLICA isteklerinde metadata yazimi atlanir (veri zaten ortak DB'de mevcut). Kullanici metadata'si (`x-amz-meta-*`) korunur. |

### v0.3.0 Yeni Ozellikler

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| S3 Select (SQL on Objects) | **Uygulanmis** | CSV ve JSON nesneler uzerinde SQL sorgulama. SELECT/WHERE/LIMIT destegi. |
| Nesne Sikistirma (zstd) | **Uygulanmis** | PUT sirasinda otomatik zstd sikistirma, GET sirasinda seffaf acma. Akilli icerik-tipi atlama. |
| Veri Tekilestirme (Dedup) | **Uygulanmis** | SHA-256 tabanli icerik adreslemeli depolama. Referans sayimi ile paylasimli depolama. |
| Degistirilemez Denetim Logu | **Uygulanmis** | Blockchain tarzi hash zinciri ile denetim gunlugu. Zincir butunluk dogrulama endpoint'i. |
| Gercek Zamanli Degisiklik Akisi (SSE) | **Uygulanmis** | Server-Sent Events ile nesne degisiklik bildirimleri. Global ve bucket bazli kanallar. |
| Gecici Bucket Paylasimi | **Uygulanmis** | Token tabanli halka acik bucket erisimi. Okuma/yazma/listeleme izinleri, suresi dolma, indirme limiti. |
| Nesne Donusum Boru Hatti | **Uygulanmis** | Asenkron goruntu isleme: thumbnail, EXIF temizleme, format donusumu. |
| Erasure Coding (Reed-Solomon) | **Uygulanmis** | Veri parcalari + parite parcalari ile veri koruma. k-of-n shard ile yeniden yapilandirma. |
| PostgreSQL Hata Gunlugu | **Uygulanmis** | 9 onceki stub metodun tam uygulamasi. Hata sorgulama, filtreleme, temizleme. |

### Diger

| Ozellik | Durum | Notlar |
|---------|-------|--------|
| `GetBucketEncryption` | **Planlanmis** | Bucket varsayilan sifreleme yapilandirmasi. |
| `PutBucketEncryption` | **Planlanmis** | Bucket varsayilan sifreleme ayarlama. |
| `DeleteBucketEncryption` | **Planlanmis** | Bucket sifreleme yapilandirmasini silme. |
| `GetBucketReplication` | **Planlanmis** | S3 uyumlu replikasyon yapilandirmasi. |
| `PutBucketReplication` | **Planlanmis** | S3 uyumlu replikasyon ayarlama. |
| `DeleteBucketReplication` | **Planlanmis** | S3 uyumlu replikasyon silme. |
| `GetBucketLogging` | **Planlanmis** | Erisim gunu tutma yapilandirmasi. |
| `PutBucketLogging` | **Planlanmis** | Erisim gunu ayarlama. |
| `GetBucketAccelerateConfiguration` | **Uygulanmayacak** | Transfer hizlandirma AWS'ye ozgu. |
| `GetBucketRequestPayment` | **Uygulanmayacak** | Istek odeme AWS'ye ozgu. |
| `GetBucketWebsite` | **Planlanmis** | Statik web sitesi barindirma. |
| `PutBucketWebsite` | **Planlanmis** | Statik web sitesi ayarlama. |
| `DeleteBucketWebsite` | **Planlanmis** | Statik web sitesi silme. |

---

## 17. Uyumluluk Ozeti

| Kategori | Uygulanmis | Kismi | Planlanmis | Toplam |
|----------|:----------:|:-----:|:----------:|:------:|
| Bucket Operasyonlari | 5 | 0 | 1 | 6 |
| Nesne Operasyonlari | 9 | 0 | 1 | 10 |
| Multipart Upload | 6 | 0 | 1 | 7 |
| Versiyonlama | 4 | 0 | 1 | 5 |
| Yasam Dongusu | 4 | 0 | 2 | 6 |
| Bucket Etiketleme | 3 | 0 | 0 | 3 |
| Nesne Etiketleme | 3 | 0 | 0 | 3 |
| CORS | 5 | 0 | 0 | 5 |
| Bucket ACL | 2 | 0 | 0 | 2 |
| Nesne ACL | 2 | 0 | 0 | 2 |
| Bucket Policy | 3 | 0 | 1 | 4 |
| Object Lock / Retention / Legal Hold | 8 | 0 | 0 | 8 |
| Bildirim | 5 | 0 | 1 | 6 |
| Presigned URL'ler | 7 | 0 | 2 | 9 |
| SSE-S3 | 2 | 1 | 1 | 4 |
| SSE-C | 2 | 1 | 0 | 3 |
| SSE-KMS | 0 | 0 | 3 | 3 |
| AWS Signature V4 | 5 | 0 | 1 | 6 |
| LDAP / Active Directory | 7 | 0 | 0 | 7 |
| Erisim Anahtari Yonetimi | 4 | 0 | 1 | 5 |
| **TOPLAM** | **85** | **2** | **16** | **103** |

---

## 18. Bilinen Kisitlamalar

1. **Tek depolama sinifi:** Yalnizca STANDARD depolama sinifi desteklenmektedir. Glacier, Intelligent-Tiering gibi siniflar bulunmamaktadir.
2. **KMS entegrasyonu yok:** `aws:kms` sifreleme basligi kabul edilmekte ancak gercek bir KMS servisiyle entegrasyon bulunmamaktadir.
3. **PostgreSQL backend:** Cluster dugum operasyonlari ve replikasyon kurallari artik PostgreSQL backend'de tam desteklenmektedir. CORS, ACL, notification, policy, object lock, retention, legal hold islemleri de PostgreSQL'de uygulanmistir.
4. **Chunked upload imzalama:** `aws-chunked` content-encoding ile parca parca imza dogrulama henuz desteklenmemektedir.
5. **STS (Security Token Service):** Gecici kimlik bilgileri ve rol tabanli erisim henuz desteklenmemektedir.
6. **Transfer Acceleration:** AWS CloudFront'a ozgu bir ozellik oldugu icin desteklenme plani yoktur.
7. **Bucket sahipligi kontrolleri:** `BucketOwnerEnforced` gibi gelismis sahiplik kontrolleri henuz uygulanmamistir.

---

## 19. Test ve Dogrulama

### Birim Testleri

```bash
# Tum birim testleri
cargo test --all

# Belirli bir crate testi
cargo test -p hafiz-s3-api
```

### S3 API Entegrasyon Testleri (52 test)

3 dugumlu kumeleme uzerinde AWS CLI v2 ile test edilmistir. Test scripti: `deploy/distributed/tests/run-tests.sh`

| Kategori | Test Sayisi | Durum |
|----------|:-----------:|:-----:|
| Bucket Operasyonlari | 4 | 4/4 PASS |
| Nesne Operasyonlari | 15 | 15/15 PASS |
| Multipart Upload | 8 | 8/8 PASS |
| Versiyonlama | 7 | 7/7 PASS |
| Bucket Etiketleme | 4 | 4/4 PASS |
| Nesne Etiketleme | 4 | 4/4 PASS |
| Lifecycle | 4 | 4/4 PASS |
| CORS | 3 | 3/3 PASS |
| SSE-C Sifreleme | 2 | 2/2 PASS |
| Temizlik ve Silme | 1 | 1/1 PASS |
| **Toplam** | **52** | **52/52 PASS** |

```bash
# Test calistirma
cd deploy/distributed/tests
bash run-tests.sh
```

### Replikasyon Testleri (29 test)

3 dugumlu kume uzerinde capraz node dogrulama. Test scripti: `deploy/distributed/tests/replication-tests.sh`

| Kategori | Test Sayisi | Durum |
|----------|:-----------:|:-----:|
| Kucuk dosya replikasyonu | 3 | 3/3 PASS |
| 1MB dosya replikasyonu + butunluk | 3 | 3/3 PASS |
| 10MB multipart replikasyonu + butunluk | 3 | 3/3 PASS |
| MD5 capraz node tutarliligi | 1 | 1/1 PASS |
| Uzerine yazma yayilimi | 5 | 5/5 PASS |
| Silme yayilimi | 6 | 6/6 PASS |
| Metadata replikasyonu | 3 | 3/3 PASS |
| Yuk dengeleyici gidis-donus | 4 | 4/4 PASS |
| **Toplam** | **29** | **29/29 PASS** |

```bash
# Replikasyon testlerini calistirma
cd deploy/distributed/tests
bash replication-tests.sh
```

### Test Ortami

| Bilesen | Detay |
|---------|-------|
| Dugum Sayisi | 3 (10.50.0.61, 10.50.0.62, 10.50.0.63) |
| Metadata Backend | PostgreSQL (paylasimli, Node1 uzerinde) |
| Yuk Dengeleyici | HAProxy (source balancing, Node1 uzerinde) |
| Test Araci | AWS CLI v2 |
| Kimlik Bilgileri | hafizadmin / HafizSecret2024! |

AWS SDK'lari (boto3, aws-sdk-js, aws-sdk-go, aws-sdk-rust) ile tam uyumluluk saglanmaktadir.

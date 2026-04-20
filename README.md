# WhatsApp Cloud API — Sesli Arama (Calling API) Rehberi

WhatsApp Cloud API'nin **Calling** özelliği, Meta iş hesaplarına gelen sesli aramaları sunucunuz üzerinden yönetmenize ve WebRTC üzerinden tarayıcıda cevaplamanıza olanak tanır. Bu rehber özelliğin uçtan uca nasıl çalıştığını anlatır — mimari, akış, handshake detayları ve Meta panel ayarları.

> Bu doküman Meta'nın resmi Cloud API'si için geçerlidir. whatsapp-web.js / Baileys gibi resmi olmayan kütüphanelerin arama desteği yoktur.
>
> **Kapsam:** Bu rehber yalnızca **gelen aramaları** (incoming calls) ele alır. Giden aramalar (outbound calls — yani sizin WhatsApp numarasından bir müşteriyi aramanız) için ayrı bir rehber ileride yazılacaktır.

---

## İçindekiler

- [Sistem Mimarisi](#sistem-mimarisi)
- [Ön Koşullar](#ön-koşullar)
- [Meta Panelinde Yapılması Gerekenler](#meta-panelinde-yapılması-gerekenler)
- [Webhook Akışı](#webhook-akışı)
- [SDP / WebRTC Handshake](#sdp--webrtc-handshake)
- [Graph API Call Kontrol Aksiyonları](#graph-api-call-kontrol-aksiyonları)
- [Birden Fazla Operatörü Yönetme](#birden-fazla-operatörü-yönetme)
- [Sık Karşılaşılan Sorunlar](#sık-karşılaşılan-sorunlar)
- [Giden Aramalar](#giden-aramalar)

---

## Sistem Mimarisi

Üç aktör vardır: **Meta Cloud API**, **Backend** (sizin sunucunuz), **Frontend** (operatörün tarayıcısı).

```
  Arayan Kişi             Meta Cloud            Backend                  Frontend
  (WhatsApp)              (WhatsApp API)        (sunucu)                 (tarayıcı)
      │                       │                     │                        │
      │── arama başlatır ────►│                     │                        │
      │                       │── webhook ─────────►│                        │
      │                       │   (ringing+SDP)     │                        │
      │                       │                     │── socket bildirimi ───►│
      │                       │                     │                        │
      │                       │                     │◄──── accept komutu ────│
      │                       │                     │      (answer SDP)      │
      │                       │                     │                        │
      │                       │◄── pre_accept ──────│                        │
      │                       │◄── accept ──────────│                        │
      │                       │                                              │
      │◄═══════ WebRTC ses akışı (peer-to-peer) ════════════════════════════►│
      │                                                                      │
      │                       │── webhook: connect ►│── bağlandı bildirimi ─►│
      │                       │                                              │
      │── kapatır ───────────►│                                              │
      │                       │── webhook: terminate│── kapandı bildirimi ──►│
```

**Kilit nokta:** Ses verisi Meta sunucularından geçmez. Arayan kişi ile operatörün tarayıcısı arasında **peer-to-peer WebRTC** kurulur. Meta sadece sinyalleşmeyi (SDP offer/answer alışverişini) taşır.

Bu yüzden bant genişliği maliyeti Meta'ya değil, arayan ve operatörün kendi internet bağlantılarına aittir. Meta sadece orkestrasyon yapar.

---

## Ön Koşullar

- Meta Business hesabı + onaylanmış WhatsApp Business numarası
- `whatsapp_business_messaging` ve `whatsapp_business_management` izinli bir **System User Access Token**
- Public HTTPS sunucusu (webhook endpoint'i için — geliştirmede ngrok/Cloudflare Tunnel iş görür)
- Modern tarayıcı (WebRTC desteği — Chrome, Firefox, Safari güncel sürümler)
- STUN/TURN server erişimi (uzak ülkelerden gelen aramalarda TURN şart olur)

---

## Meta Panelinde Yapılması Gerekenler

Calling özelliğinin çalışması için Meta tarafında **iki ayrı ayar** gereklidir. İkisi de yapılmazsa arama butonu görünmez veya webhook gelmez.

### 1. Webhook Abonelikleri

Meta App Dashboard → WhatsApp → Configuration:

- **Callback URL:** sunucunuzun webhook endpoint'i (örn. `https://sizin-sunucu.com/webhook`)
- **Verify token:** kendi seçtiğiniz rastgele string
- **Webhook fields:** `messages` ve `calls` alanlarının ikisi de abone edilmelidir. Sadece `messages` aboneliği yeterli değildir — calls event'leri ayrı bir field olarak düşer.

### 2. Telefon Numarası Seviyesinde Arama Özelliğini Açma

**Bu en çok atlanan adımdır.** Webhook'u doğru kursanız bile numaranın kendi ayarlarında arama özelliği açık değilse Meta webhook göndermez ve kullanıcı WhatsApp'ta arama butonunu bile göremez.

Meta Business Manager → WhatsApp Manager → Phone Numbers → **numaranızı seçin** → Settings → **Calling** sekmesi.

Burada:

- **Calling status:** `Enabled` yapılmalı
- **Call icon visibility:** `Default` (müşteri WhatsApp sohbetinde arama butonunu görsün)
- **Business hours:** isteğe bağlı olarak çalışma saati dışında otomatik reddet ayarlanabilir

Bu ayarı Graph API üzerinden de değiştirebilirsiniz (phone number settings endpoint'i), ama ilk sefer panelden yapmak hem daha hızlı hem Meta'nın calling review sürecini tetikler.

**Not:** 2025 itibarıyla Calling API hâlâ limited availability'de. Meta App'iniz calling whitelisting'den geçmemişse ayar bölümü kapalı görünebilir — bu durumda Meta destek kanalından başvurmanız gerekir.

---

## Webhook Akışı

Meta, bir aramanın yaşam döngüsü boyunca üç farklı event gönderir. Hepsi aynı webhook URL'inize düşer, `field: "calls"` ile gelirler.

### Event 1 — `ringing` (veya `initiated`)

Birisi numaranızı aradığında tetiklenir. Payload içeriğinde önemli olanlar:

- **Call ID** — aramanın benzersiz kimliği. Sonraki tüm işlemlerde (accept, reject, terminate) bu ID'yi kullanacaksınız
- **From** — arayanın telefon numarası
- **Caller profile name** — arayanın WhatsApp profil ismi
- **Phone number ID** — hangi numaranızın arandığı (birden fazla numaranız varsa yönlendirme için)
- **Session SDP** — WebRTC offer metni. Aramayı kabul ederseniz buna karşılık bir answer üretmeniz gerekir
- **Timestamp** — aramanın başladığı zaman

Bu event'i aldığınızda hızlıca frontend'e bildirmelisiniz. Meta, cevap almak için yaklaşık 30 saniye bekler, sonra kendiliğinden timeout'a düşer ve `terminate` gönderir.

### Event 2 — `connect`

Accept komutunu Meta'ya gönderdikten sonra Meta taraf bağlantıyı kurduğunda düşer. Bu aşamada ses akışı başlamıştır. `session.sdp` güncellenmiş halde tekrar gelebilir — ICE renegotiation için referans.

### Event 3 — `terminate`

Arama herhangi bir sebeple bittiğinde düşer. `status` alanı sonucu verir:

- `COMPLETED` — normal kapanış
- `CANCELED` — arayan cevap beklemeden kapattı
- `NO_ANSWER` — timeout (kimse cevaplamadı)
- `FAILED` — teknik hata
- `REJECTED` — siz reddettiniz

Bu event geldiğinde tarayıcıdaki WebRTC bağlantısını ve mikrofon stream'ini temizlemeniz gerekir.

---

## SDP / WebRTC Handshake

Meta **offer** gönderir, siz **answer** üretmek zorundasınız. Kritik ve belgelenmemiş bir detay var:

**Meta'nın offer SDP'sinde `a=setup:actpass` satırı bulunur. Gönderdiğiniz answer'da bu değeri `a=setup:active` olarak değiştirmezseniz DTLS el sıkışması başarısız olur ve arama 5 saniye içinde otomatik koparır.**

Bu detay Meta'nın resmi dokümanlarında yok. Setup attribute'ü DTLS-SRTP'nin hangi tarafın aktif rol alacağını belirler; Meta sunucusu aramalarda pasif rol beklediği için siz active rolü üstlenmek zorundasınız.

Handshake adımları:

1. Tarayıcıda mikrofon izni al (`getUserMedia`)
2. `RTCPeerConnection` kur, mikrofon track'ini ekle
3. Meta'dan gelen SDP'yi remote description olarak set et
4. `createAnswer` çağır
5. Answer SDP'sindeki `setup:actpass` ifadesini `setup:active` ile değiştir
6. Düzeltilmiş SDP'yi local description olarak set et ve backend üzerinden Meta'ya gönder

Meta answer'ı aldıktan sonra STUN üzerinden peer discovery tamamlanır, DTLS anahtar değişimi yapılır ve SRTP ses kanalı açılır.

---

## Graph API Call Kontrol Aksiyonları

Backend'iniz Meta'ya arama kontrol komutlarını `POST /v21.0/{PHONE_NUMBER_ID}/calls` endpoint'ine gönderir. Action tipleri:

### `pre_accept`

Accept sürecinin ilk adımı. Answer SDP'sini taşır. Meta bunu ICE candidate gathering için kullanır.

### `accept`

`pre_accept`'ten yaklaşık 1 saniye sonra aynı SDP ile gönderilir. Bu iki aşamalı yapı Meta'nın sunucu tarafında media relay hazırlamasına zaman tanır. Tek seferlik `accept` göndermek `invalid session` hatası verir.

### `reject`

Aramayı cevaplamadan reddeder. Karşı taraf WhatsApp'ta "meşgul" tonu duyar.

### `terminate`

Açık bir aramayı kapatır. Sadece bağlantı kurulmuş (accepted/connected) aramalar için kullanılır.

Tüm komutların başarılı olması için request header'ında `Authorization: Bearer {ACCESS_TOKEN}` bulunmalıdır. Response 200 değilse genellikle `error.message` alanında sebep açıklanır.

---

## Birden Fazla Operatörü Yönetme

Call center senaryosunda aynı numaranın gelen aramalarını birden fazla operatör aynı anda görür. İlk kabul eden alır, diğerlerinde buton pasifleşir.

### Race Condition Problemi

Webhook → Socket.IO → Frontend gecikmesi ~200-500ms civarıdır. İki operatör neredeyse aynı anda "Kabul" butonuna basarsa ikisi de Meta'ya accept gönderebilir. İkincisi `invalid session` hatası alır ama UX açısından kötü görünür.

### Çözüm

Backend'de her call için bir durum atomiği tutulmalı. İlk accept isteği geldiğinde durum `ringing → accepting` olarak kilitlenir, sonraki accept istekleri reddedilir. Kilit başarılı olduğunda diğer operatörlere "X operatörü tarafından cevaplandı" bildirimi broadcast edilir ve UI otomatik güncellenir.

### Kimin Cevapladığını Gösterme

Kabul eden operatörün kullanıcı ID'si `callHandledBy` gibi bir event ile tüm abonelere yayınlanır. Diğer operatörlerin ekranında o kişinin sohbeti üzerinde "Ahmet Yılmaz aramada" rozeti belirir. Aynı mantık reddetme için de kurulur.

### Kendi Takip Ettiğiniz Durumlar

Bellekte tutmanız gereken minimum alanlar:
- Call ID
- Phone Number ID
- Arayan numara
- Durum (ringing, accepting, accepted, connected, terminated)
- Kabul eden kullanıcı ID'si (varsa)
- Remote SDP (accept akışında gerekli)
- Kabul zamanı (süre hesaplamak için)

Bu bilgiler genelde bellekte Map halinde tutulur; kalıcı bir şey gerekmez çünkü arama bittikten sonra ihtiyaç kalmaz. Sadece "arama log'u" olarak veritabanına bir mesaj satırı atılması yeterlidir.

---

## Sık Karşılaşılan Sorunlar

| Belirti | Muhtemel Sebep |
|---|---|
| Arama 5 saniye sonra otomatik kopuyor | Answer SDP'de `setup:actpass` kalmış; `setup:active` yapılmalı |
| Webhook hiç gelmiyor | Meta panelinden `calls` field'ı subscribe edilmemiş, veya HTTPS sertifikası geçersiz |
| WhatsApp arama butonu görünmüyor | Phone Number settings'te Calling status `Enabled` değil |
| `accept` response 400: invalid session | Önce `pre_accept` göndermeyi unutmuşsunuz; sıra önemli |
| Mikrofon sesi gidiyor ama karşı taraf duyulmuyor | `ontrack` event handler'ı remote stream'i audio element'ine bağlamıyor |
| Uzak ülkelerden gelen aramalarda bağlantı kurulmuyor | STUN yetersiz; kendi TURN sunucunuzu kurmalısınız (coturn) |
| `Calling permission not enabled` hatası | Meta App calling review'ndan geçmemiş; beta listesine başvurun |
| Webhook geliyor ama call ID sonraki request'te bulunamıyor | Backend yeniden başlatılmış ve bellek silinmiş; Map'e state restore eklenmeli |

---

## Giden Aramalar

Bu rehber yalnızca **gelen aramaları** kapsar. Bir müşteriyi WhatsApp numaranızdan arama (outbound call / giden arama) akışı farklı çalışır: SDP offer'ı siz üretirsiniz, Meta'ya önce `initiate` komutu gönderirsiniz, karşı taraf WhatsApp'ta aramayı görür ve kabul ederse answer SDP'si webhook ile size döner. Ayrıca giden aramalar Meta tarafında kota + faturalama kurallarına tabidir.

**Giden aramalar için ayrı bir rehber ileride yazılacaktır.**

---

## Kaynaklar

- [WhatsApp Cloud API Calling Docs](https://developers.facebook.com/docs/whatsapp/cloud-api/calling) — resmi
- [WebRTC Getting Started](https://webrtc.org/getting-started/overview) — temel kavramlar
- [Meta Graph API Reference](https://developers.facebook.com/docs/graph-api) — endpoint detayları
- [coturn TURN Server](https://github.com/coturn/coturn) — TURN sunucusu kurmak için

---

## Lisans

MIT

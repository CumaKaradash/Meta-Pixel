# Meta Pixel & Conversions API Integrated Master Guide (2026)

> **Last Updated / Son Güncelleme:** August 2026 / Ağustos 2026

**🇬🇧 Available in English | 🇹🇷 Türkçe Olarak Mevcut**

---

## Quick Navigation / Hızlı Navigasyon

- [🇬🇧 **ENGLISH VERSION** / İngilizce Versiyon](#english-version)
- [🇹🇷 **TURKISH VERSION** / Türkçe Versiyon](#turkish-version)

---

<a id="english-version"></a>

# 🇬🇧 ENGLISH VERSION

## Table of Contents

1. [Introduction and 1:1 Matching Principle](#1-introduction-and-11-matching-principle)
2. [Meta Events Manager Configuration Settings](#2-meta-events-manager-configuration-settings)
3. [Hybrid Architecture (Pixel + CAPI) and Deduplication](#3-hybrid-architecture-pixel--capi-and-deduplication)
4. [Event Match Quality (EMQ) Score (8.5+ Score Guide)](#4-event-match-quality-emq-score-85-score-guide)
5. [Lead Forms: 1:1 Matching Verification Steps](#5-lead-forms-11-matching-verification-steps)
6. [Standard and Custom Event Parameter Structure](#6-standard-and-custom-event-parameter-structure)
7. [Complete Code Implementations (JS, Node.js, Python, PHP)](#7-complete-code-implementations-js-nodejs-python-php)
8. [CMS and E-Commerce Platform Settings](#8-cms-and-e-commerce-platform-settings)
9. [Testing, Audit, and Debugging](#9-testing-audit-and-debugging)
10. [GDPR and Privacy Compliance](#10-gdpr-and-privacy-compliance)

---

## 1. Introduction and 1:1 Matching Principle

One of the most common issues in digital marketing campaigns is the **inconsistency between the number of leads/conversions shown in Meta Ads Manager and the actual customer records in the database**.

### Why Is 100% Matching Impossible Without Proper Setup?

- **Safari 26 / iOS 26 Advanced Fingerprinting Protection (AFP)**
  - Removal of `fbclid` parameters from email and message links
  - Impact: 25-40% data loss on iOS devices

- **Ad Blockers and ITP Restrictions**
  - Browser-based JavaScript Pixel code is blocked
  - Causes 25-40% tracking loss

- **Spam Form Submissions / Bot Fills**
  - Client-side button clicks trigger events that never reach the database
  - Creates false lead counts in Meta

- **Missing Deduplication**
  - Running both Pixel and CAPI without proper `event_id` matching
  - Results in double counting of events

### 2026 Standard Solution

Meta Pixel (Client-Side) and Conversions API (Server-Side) must be combined in a **hybrid architecture**, with CAPI signals sent **only for database-verified successful submissions** using matching `event_id` values. This achieves **100% (1:1) accurate data matching** and **average 17.8% lower acquisition cost (CPA)**.

---

## 2. Meta Events Manager Configuration Settings

Critical settings in Meta Business Manager and Events Manager for proper data tracking:

### Step 1: Dataset and Pixel ID Creation

1. Meta Business Suite → **Events Manager** panel
2. **Connect Data Sources** → Select **Web**
3. Name your dataset with your domain (Example: `domain.com-dataset`)
4. Copy the generated **Dataset ID / Pixel ID**

### Step 2: Conversions API Access Token Generation

1. Events Manager → Your Dataset → **Settings** tab
2. Scroll to **Conversions API** section
3. Click **Generate access token**
4. Copy the long alphanumeric token and store securely in `.env` file (*This token will not be shown again!*)

### Step 3: Automatic Advanced Matching

1. In **Settings** tab, enable **Automatic Advanced Matching**
2. Activate all customer information parameters:
   - Email (`em`)
   - Phone Number (`ph`)
   - First Name (`fn`) and Last Name (`ln`)
   - City (`ct`), State (`st`), Postal Code (`zp`), Country (`country`)
   - External ID (`external_id`)

### Step 4: Aggregated Event Measurement (AEM) & Domain Verification

1. Business Settings → **Brand Safety → Domains** to verify your website
2. In Aggregated Event Measurement, prioritize **top 8 conversion events** for iOS
   - Highest priority: `Purchase` / `Lead`

---

## 3. Hybrid Architecture (Pixel + CAPI) and Deduplication

When the same event comes from both browser and server, **Event Deduplication** prevents double counting.

```
            ┌─────────────────────────────────────────────┐
            │        Customer Form / Application           │
            └────────────────────┬────────────────────────┘
                                 │
                ┌────────────────┴────────────────┐
                ▼                                 ▼
    ┌──────────────────────┐        ┌──────────────────────┐
    │ Browser (Client-Side)│        │ Server (Server-Side) │
    │   Meta Pixel JS      │        │ Database-Verified    │
    │                      │        │       CAPI           │
    │ event_name: "Lead"   │        │ event_name: "Lead"   │
    │ event_id: "lead_..." │        │ event_id: "lead_..." │
    └──────────┬───────────┘        └──────────┬───────────┘
               │                                │
               └────────────────┬───────────────┘
                                │
                                ▼
                 ┌─────────────────────────────┐
                 │   Meta Events Manager       │
                 │ event_id & event_name Match │
                 │  ──► 1:1 SINGLE CONVERSION ◄──
                 └─────────────────────────────┘
```

### Deduplication Rules Checklist

- ✅ **`event_name` Must Match Exactly** - Case-sensitive identical text (e.g., `"Lead"` or `"Purchase"`)
- ✅ **`event_id` Must Be Unique and Identical** - Server-generated, passed to both client and server (e.g., `"lead_2026_8839"`)
- ✅ **Time Window** - Client and server events must reach Meta within 48 hours

---

## 4. Event Match Quality (EMQ) Score (8.5+ Score Guide)

EMQ score measures Meta's success in matching your conversion data to actual Facebook/Instagram users (0-10 scale).

### Parameters Required for 8.5+ EMQ Score

| Parameter | Description | Format / Encryption |
|-----------|-------------|---------------------|
| `em` | Email Address | Lowercase, no spaces → **SHA-256** |
| `ph` | Phone Number | With country code (e.g., 905462388707) → **SHA-256** |
| `fn` / `ln` | First Name / Last Name | Lowercase → **SHA-256** |
| `ct` / `st` / `zp` | City / State / Postal Code | Lowercase (e.g., `ankara`) → **SHA-256** |
| `country` | Country Code | 2-digit ISO code (e.g., `tr`) → **SHA-256** |
| `fbp` | Meta Browser Cookie | `_fbp` cookie value (Plain text) |
| `fbc` | Meta Click Cookie | `_fbc` or `fbclid` from URL (Plain text) |
| `external_id` | Database User/Customer ID | Unique ID → **SHA-256** |
| `client_ip_address` | User IP Address | IPv4 or IPv6 (Plain text) |
| `client_user_agent` | Browser User Agent | `navigator.userAgent` (Plain text) |

---

## 5. Lead Forms: 1:1 Matching Verification Steps

**1:1 Matching Checklist** to ensure Meta Ads reports match actual database records:

1. **Don't Trigger "Lead" on Button Click**
   - Only trigger after form validation passes
   - Wait for **server success response**
   - Client Pixel fires only in success state

2. **Trigger CAPI on Server Side**
   - Send CAPI `Lead` event immediately when data is successfully written to database

3. **Generate Static Submission ID**
   - Server generates unique `form_submission_id` at submission time
   - Return to client for Pixel `eventID` parameter
   - Include in CAPI payload as `event_id`

4. **Spam & Bot Filtering**
   - Use reCAPTCHA or Cloudflare Turnstile validation
   - Don't fire Pixel or CAPI for failed validations

---

## 6. Standard and Custom Event Parameter Structure

### Standard Lead Event Parameter Schema

```json
{
  "event_name": "Lead",
  "event_id": "lead_id_99812",
  "custom_data": {
    "lead_type": "Web Design Quote Form",
    "value": 1500.00,
    "currency": "USD"
  }
}
```

### Standard Events Reference

| Event | Description | Use Case | Required Parameters |
|-------|-------------|----------|---------------------|
| `PageView` | Page view | All pages | - |
| `ViewContent` | Product detail view | Product page | `content_ids`, `value` |
| `Search` | Site search | Search results | `search_string` |
| `AddToCart` | Add to cart | Cart action | `content_ids`, `value` |
| `InitiateCheckout` | Start checkout | Checkout page | `content_ids`, `num_items` |
| `Purchase` | Complete purchase | Order confirmation | `content_ids`, `value`, `num_items` |
| `Lead` | Form submission | Contact/Quote form | `value` (optional) |
| `CompleteRegistration` | Registration complete | Sign-up page | `status` |

---

## 7. Complete Code Implementations (JS, Node.js, Python, PHP)

### 7.1. Client-Side (HTML/JS Base Code & Advanced Matching)

Add this to the `<head>` of all pages:

```html
<!-- Meta Pixel Base Code -->
<script>
!function(f,b,e,v,n,t,s){if(f.fbq)return;n=f.fbq=function(){n.callMethod?
n.callMethod.apply(n,arguments):n.queue.push(arguments)};
if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
n.queue=[];t=b.createElement(e);t.async=!0;
t.src=v;s=b.getElementsByTagName(e)[0];
s.parentNode.insertBefore(t,s)}(window, document,'script',
'https://connect.facebook.net/en_US/fbevents.js');

fbq('init', 'YOUR_DATASET_ID');
fbq('track', 'PageView');
</script>

<noscript>
  <img height="1" width="1" style="display:none"
       src="https://www.facebook.com/tr?id=YOUR_DATASET_ID&ev=PageView&noscript=1" />
</noscript>

<!-- Form Success Handler -->
<script>
function handleFormSuccess(responseData) {
  // responseData.submissionId is the unique submission ID from server
  fbq('track', 'Lead', {
    content_name: 'Contact Form',
    value: 500.00,
    currency: 'USD'
  }, {
    eventID: responseData.submissionId // Deduplication ID
  });
}
</script>
```

---

### 7.2. Node.js Server-Side (CAPI + SHA-256 Hashing)

```javascript
const crypto = require('crypto');
const axios = require('axios');

// SHA-256 Hash Function
function hashData(data) {
  if (!data) return undefined;
  return crypto.createHash('sha256')
    .update(data.trim().toLowerCase())
    .digest('hex');
}

async function sendCapiLead({
  datasetId,
  accessToken,
  submissionId,
  eventSourceUrl,
  userData,
  customData
}) {
  
  const url = `https://graph.facebook.com/v25.0/${datasetId}/events`;

  const payload = {
    data: [{
      event_name: 'Lead',
      event_time: Math.floor(Date.now() / 1000),
      event_id: submissionId, // Deduplication ID
      event_source_url: eventSourceUrl,
      action_source: 'website',
      user_data: {
        em: userData.email ? [hashData(userData.email)] : undefined,
        ph: userData.phone ? [hashData(userData.phone)] : undefined,
        fn: userData.firstName ? [hashData(userData.firstName)] : undefined,
        ln: userData.lastName ? [hashData(userData.lastName)] : undefined,
        client_ip_address: userData.ipAddress,
        client_user_agent: userData.userAgent,
        fbp: userData.fbp,
        fbc: userData.fbc
      },
      custom_data: customData
    }],
    access_token: accessToken
  };

  try {
    const res = await axios.post(url, payload);
    console.log('✅ Lead CAPI sent successfully:', res.data);
    return res.data;
  } catch (err) {
    console.error('❌ CAPI Error:', err.response ? err.response.data : err.message);
  }
}

// Example Usage
sendCapiLead({
  datasetId: 'YOUR_DATASET_ID',
  accessToken: 'YOUR_ACCESS_TOKEN',
  submissionId: 'lead_2026_8839',
  eventSourceUrl: 'https://example.com/contact',
  userData: {
    email: 'john@example.com',
    phone: '+19175551234',
    firstName: 'John',
    lastName: 'Doe',
    ipAddress: '185.190.10.1',
    userAgent: 'Mozilla/5.0...',
    fbp: 'fb.1.1711000000.123456789',
    fbc: 'fb.1.1711000000.IwAR0...'
  },
  customData: {
    value: 500.00,
    currency: 'USD',
    lead_type: 'Contact Form'
  }
});
```

---

### 7.3. Python Server-Side (CAPI + Deduplication)

```python
import time
import hashlib
import requests

def sha256_hash(val: str) -> str:
    if not val:
        return None
    return hashlib.sha256(
        val.strip().lower().encode('utf-8')
    ).hexdigest()

def send_capi_lead(dataset_id: str, access_token: str, submission_data: dict):
    endpoint = f"https://graph.facebook.com/v25.0/{dataset_id}/events"

    payload = {
        "data": [{
            "event_name": "Lead",
            "event_time": int(time.time()),
            "event_id": submission_data["submission_id"],  # Deduplication ID
            "event_source_url": submission_data["url"],
            "action_source": "website",
            "user_data": {
                "em": [sha256_hash(submission_data["email"])],
                "ph": [sha256_hash(submission_data["phone"])],
                "fn": [sha256_hash(submission_data["first_name"])],
                "ln": [sha256_hash(submission_data["last_name"])],
                "client_ip_address": submission_data["ip"],
                "client_user_agent": submission_data["user_agent"],
                "fbp": submission_data.get("fbp"),
                "fbc": submission_data.get("fbc")
            },
            "custom_data": {
                "value": 500.00,
                "currency": "USD",
                "lead_type": submission_data.get("lead_type")
            }
        }],
        "access_token": access_token
    }

    res = requests.post(endpoint, json=payload)
    return res.json()

# Example Usage
if __name__ == "__main__":
    result = send_capi_lead(
        dataset_id="YOUR_DATASET_ID",
        access_token="YOUR_ACCESS_TOKEN",
        submission_data={
            "submission_id": "lead_2026_8839",
            "email": "john@example.com",
            "phone": "19175551234",
            "first_name": "John",
            "last_name": "Doe",
            "ip": "185.190.10.1",
            "user_agent": "Mozilla/5.0...",
            "url": "https://example.com/contact",
            "lead_type": "Contact Form"
        }
    )

    print("Meta API Response:", result)
```

---

### 7.4. PHP Form Submission (Lead CAPI Integration)

```php
<?php

function send_meta_capi_lead($dataset_id, $access_token, $data) {
    $url = "https://graph.facebook.com/v25.0/{$dataset_id}/events";

    $user_data = [
        'em' => [hash('sha256', strtolower(trim($data['email'])))],
        'ph' => [hash('sha256', preg_replace('/[^0-9]/', '', $data['phone']))],
        'fn' => [hash('sha256', strtolower(trim($data['first_name'])))],
        'ln' => [hash('sha256', strtolower(trim($data['last_name'])))],
        'client_ip_address' => $_SERVER['REMOTE_ADDR'],
        'client_user_agent' => $_SERVER['HTTP_USER_AGENT'],
        'fbp' => $_COOKIE['_fbp'] ?? null,
        'fbc' => $_COOKIE['_fbc'] ?? null
    ];

    $payload = [
        'data' => [
            [
                'event_name' => 'Lead',
                'event_time' => time(),
                'event_id' => $data['submission_id'], // Deduplication ID
                'event_source_url' => $data['page_url'],
                'action_source' => 'website',
                'user_data' => $user_data,
                'custom_data' => [
                    'value' => 500.00,
                    'currency' => 'USD',
                    'lead_type' => $data['lead_type'] ?? 'Contact Form'
                ]
            ]
        ],
        'access_token' => $access_token
    ];

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

    $response = curl_exec($ch);
    curl_close($ch);

    return json_decode($response, true);
}

// Example Usage
$result = send_meta_capi_lead(
    'YOUR_DATASET_ID',
    'YOUR_ACCESS_TOKEN',
    [
        'submission_id' => 'lead_2026_8839',
        'email' => 'john@example.com',
        'phone' => '9175551234',
        'first_name' => 'John',
        'last_name' => 'Doe',
        'page_url' => 'https://example.com/contact',
        'lead_type' => 'Contact Form'
    ]
);

echo json_encode($result);

?>
```

---

## 8. CMS and E-Commerce Platform Settings

### Shopify

- Install **Facebook & Instagram** channel
- Set Data Sharing to **Maximum** (enables CAPI + Advanced Matching automatically)
- Deduplication and `fbp`/`fbc` matching handled by system

### Custom Platforms (İkas / Ticimax / Ideasoft)

1. Go to Integrations → **Meta Pixel & CAPI** section
2. Enter your **Dataset ID** and **Access Token**
3. Enable automatic server-side CAPI mode

### WooCommerce

- Install **PixelYourSite Pro** plugin
- Enter CAPI Access Token and Dataset ID
- Enable **Automatic Advanced Matching** and **Event ID Deduplication**

---

## 9. Testing, Audit, and Debugging

### 1. Test Events Mode

1. Events Manager → **Test Events** tab
2. Copy the `TESTXXXXX` code displayed
3. Add `"test_event_code": "TESTXXXXX"` to CAPI request
4. Submit form and verify in Events Manager:
   - **Browser** and **Server** signals match with same `event_id`
   - **"Deduplicated"** label appears

### 2. Browser Testing Tools

- **Meta Pixel Helper** Chrome Extension
  - Verify client-side Pixel fires correctly
  - Check parameters and identify errors
  - Monitor network requests

### 3. Performance Metrics Checklist

- ✅ **Event Coverage Ratio**: CAPI/Pixel ratio **≥75%**
- ✅ **EMQ Score**: Lead and Purchase events **≥8.0**
- ✅ **1:1 Matching**: Daily database records vs Meta reports difference **≤2%**

---

## 10. GDPR and Privacy Compliance

### User Consent Management

- When user denies tracking consent: call `fbq('consent', 'revoke')`
- On server-side: stop sending CAPI events if consent not granted
- Respect browser Do Not Track (DNT) settings

### SHA-256 Encryption is Mandatory

- **Never send raw PII** (email, phone, names) over the network
- Always hash with SHA-256 before transmission
- Hash locally before API calls

### Data Processing Options (EU Regulations)

- California CCPA: Set `data_processing_options` parameter
- EU GDPR: Configure data processing location
- Meta provides options for data retention and deletion

---

<a id="turkish-version"></a>

# 🇹🇷 TÜRKÇE VERSİYON

## İçindekiler

1. [Giriş ve 1:1 Eşleşme İlkesi](#1-giriş-ve-11-eşleşme-ilkesi-tr)
2. [Meta Events Manager Yapılandırma Ayarları](#2-meta-events-manager-yapılandırma-ayarları-tr)
3. [Hibrit Mimari (Pixel + CAPI) ve Çift Saymayı Önleme](#3-hibrit-mimari-pixel--capi-ve-çift-saymayı-önleme-deduplication-tr)
4. [Etkinlik Eşleşme Kalitesi (EMQ) Skoru](#4-etkinlik-eşleşme-kalitesi-event-match-quality---emq-skoru-85-skor-rehberi-tr)
5. [Lead Formlarında 1:1 Eşleşme Kontrol Adımları](#5-lead-müşteri-adayı-formlarında-11-eşleşme-kontrol-adımları-tr)
6. [Standart ve Özel Etkinlik Parametre Yapısı](#6-standart-ve-özel-etkinlik-parametre-yapısı-tr)
7. [Tam Kod Uygulamaları (JS, Node.js, Python, PHP)](#7-tam-kod-uygulamaları-js-nodejs-python-php-tr)
8. [CMS ve E-Ticaret Panel Ayarları](#8-cms-ve-e-ticaret-panel-ayarları-tr)
9. [Test, Denetim ve Hata Ayıklama](#9-test-denetim-audit-ve-hata-ayıklama-debugging-tr)
10. [KVKK, GDPR ve Gizlilik Uyumluluğu](#10-kvkk-gdpr-ve-gizlilik-uyumluluğu-tr)

---

## 1. Giriş ve 1:1 Eşleşme İlkesi {#1-giriş-ve-11-eşleşme-ilkesi-tr}

Dijital pazarlama kampanyalarında en sık karşılaşılan sorunlardan biri, **Meta Ads Panelinde görünen başvuru/lead sayısı ile veritabanına düşen gerçek müşteri sayısı arasındaki tutarsızlıktır**.

### Neden %100 Eşleşme Sağlanamıyor?

- **Safari 26 / iOS 26 Advanced Fingerprinting Protection (AFP)**
  - E-posta ve mesajlardan gelen linklerdeki `fbclid` parametrelerinin silinmesi
  - Sonuç: iOS cihazlarda %25-40 arasında veri kaybı

- **Reklam Engellenciler (AdBlockers) ve ITP Kısıtlamaları**
  - Tarayıcı tabanlı JavaScript Pixel kodunun engellenmesi
  - %25-40 arası takip kaybı

- **Spam Form Doldurmalar / Bot Başvuruları**
  - İstemci tarafında buton tıklamasıyla tetiklenen ancak veritabanına ulaşamayan geçersiz istekler
  - Meta panelinde sahte lead sayıları oluşturur

- **Eksik Deduplication**
  - Hem Pixel hem CAPI çalıştığında benzersiz `event_id` kullanılmaması
  - Verilerin iki kez sayılmasına neden olur

### 2026 Çözüm Standardı

Meta Pixel (Client-Side) ve Conversions API (Server-Side) **hibrit mimaride** birleştirilmeli, yalnızca **sunucu tarafında veritabanı onayı alan başarılı başvurular** için `event_id` ile eşleşen CAPI sinyali gönderilmelidir. Bu sayede:

- **%100 (1:1) Gerçek Veri Eşleşmesi** elde edilir
- **Ortalama %17.8 daha düşük edinme maliyeti (CPA)** sağlanır

---

## 2. Meta Events Manager Yapılandırma Ayarları {#2-meta-events-manager-yapılandırma-ayarları-tr}

Doğru veri takibi için Meta Business Manager ve Events Manager panelinde yapılması gereken kritik ayarlar:

### Adım 1: Dataset (Veri Seti) ve Pixel ID Oluşturma

1. Meta Business Suite → **Events Manager (Olay Yöneticisi)** paneline gidin
2. **Connect Data Sources (Veri Kaynaklarını Bağla)** → **Web** seçeneğini işaretleyin
3. Veri setinize alan adınızı içeren net bir isim verin
   - Örn: `example.com-dataset`
4. Üretilen **Dataset ID / Pixel ID** bilgisini kopyalayın

### Adım 2: Conversions API Access Token Üretme

1. Events Manager → İlgili Veri Seti → **Settings (Ayarlar)** sekmesi
2. **Conversions API** başlığına gidin
3. **Generate access token** bağlantısına tıklayın
4. Uzun alfa-numerik jetonu kopyalayıp `.env` dosyasında saklayın
   - ⚠️ **Bu token bir daha gösterilmez!**

### Adım 3: Automatic Advanced Matching (Otomatik Gelişmiş Eşleştirme)

1. **Ayarlar** sekmesinde **Automatic Advanced Matching** seçeneğini **AÇIK** yapın
2. Aşağıdaki müşteri bilgi parametrelerinin tamamını aktif edin:
   - E-posta (`em`)
   - Telefon Numarası (`ph`)
   - Ad (`fn`) ve Soyad (`ln`)
   - Şehir (`ct`), İlçe (`st`), Posta Kodu (`zp`), Ülke (`country`)
   - External ID (`external_id`)

### Adım 4: Aggregated Event Measurement (AEM) & Domain Doğrulama

1. Business Settings → **Brand Safety → Domains** altından web sitenizi doğrulayın
2. Aggregated Event Measurement bölümünde iOS cihazlar için en kritik **8 dönüşüm olayını önceliklendirin**
   - En yüksek öncelik: `Purchase` / `Lead`

---

## 3. Hibrit Mimari (Pixel + CAPI) ve Çift Saymayı Önleme (Deduplication) {#3-hibrit-mimari-pixel--capi-ve-çift-saymayı-önleme-deduplication-tr}

Hem tarayıcıdan hem sunucudan gelen aynı olayın çifte sayılmaması için **Event Deduplication** uygulanır.

```
            ┌─────────────────────────────────────────────┐
            │          Müşteri Formu / Başvuru            │
            └────────────────────┬────────────────────────┘
                                 │
                ┌────────────────┴────────────────┐
                ▼                                 ▼
    ┌──────────────────────┐        ┌──────────────────────┐
    │ Tarayıcı (Client)    │        │ Sunucu (Server)      │
    │  Meta Pixel JS       │        │ Veritabanı Onaylı    │
    │                      │        │     CAPI             │
    │ event_name: "Lead"   │        │ event_name: "Lead"   │
    │ event_id: "lead_..." │        │ event_id: "lead_..." │
    └──────────┬───────────┘        └──────────┬───────────┘
               │                                │
               └────────────────┬───────────────┘
                                │
                                ▼
                 ┌─────────────────────────────┐
                 │   Meta Events Manager       │
                 │ event_id & event_name Eşleş.│
                 │  ──► 1:1 TEK DÖNÜŞÜM ◄──   │
                 └─────────────────────────────┘
```

### Deduplication Kuralları Kontrol Listesi

- ✅ **`event_name` Birebir Aynı Olmalıdır** - Harf büyüklüğü dahil tamamen özdeş (Örn: `"Lead"` veya `"Purchase"`)
- ✅ **`event_id` Benzersiz ve Özdeş String** - Sunucuda oluşturulup tarayıcıya aktarılan ID (Örn: `"lead_2026_8839"`)
- ✅ **Zaman Penceresi** - İstemci ve sunucu etkinlikleri 48 saat içinde Meta'ya ulaşmalı

---

## 4. Etkinlik Eşleşme Kalitesi (Event Match Quality - EMQ) Skoru (8.5+ Skor Rehberi) {#4-etkinlik-eşleşme-kalitesi-event-match-quality---emq-skoru-85-skor-rehberi-tr}

EMQ skoru, Meta'nın gönderilen dönüşümü gerçek bir Facebook/Instagram kullanıcısı ile eşleştirme başarısını ölçer (0-10 arası puan).

### 8.5+ EMQ Skoru İçin Gönderilmesi Gereken Parametreler

| Parametre | Açıklama | Format / Şifreleme |
|-----------|----------|-------------------|
| `em` | E-posta Adresi | Küçük harf, boşluksuz → **SHA-256** |
| `ph` | Telefon Numarası | Ülke kodu dahil (Örn: 905462388707) → **SHA-256** |
| `fn` / `ln` | Ad / Soyad | Küçük harf → **SHA-256** |
| `ct` / `st` / `zp` | Şehir / İlçe / Posta Kodu | Küçük harf (Örn: `ankara`) → **SHA-256** |
| `country` | Ülke Kodu | 2 haneli ISO kodu (Örn: `tr`) → **SHA-256** |
| `fbp` | Meta Browser Çerezi | `_fbp` çerez değeri (Ham metin) |
| `fbc` | Meta Click Çerezi | `_fbc` veya URL'deki `fbclid` (Ham metin) |
| `external_id` | Veritabanı Kullanıcı ID | Benzersiz ID → **SHA-256** |
| `client_ip_address` | Kullanıcı IP Adresi | IPv4 veya IPv6 (Ham metin) |
| `client_user_agent` | Tarayıcı User Agent | `navigator.userAgent` (Ham metin) |

---

## 5. Lead (Müşteri Adayı) Formlarında 1:1 Eşleşme Kontrol Adımları {#5-lead-müşteri-adayı-formlarında-11-eşleşme-kontrol-adımları-tr}

Meta Ads raporlarındaki başvuru sayısı ile veritabanındaki gerçek kişi sayısının birebir tutması için takip edilecek **1:1 Kontrol Listesi**:

1. **İstemcide Buton Tıklamasına "Lead" Bağlamayın**
   - Sadece forma tıklandığında değil, yalnızca form validasyonu geçip **sunucu onay yanıtı döndüğünde** istemci Pixel'ini tetikleyin
   - Başarı durumunda (Success State) çalışacak şekilde yapılandırın

2. **Sunucu Tarafında CAPI Tetikleyin**
   - Form verisi veritabanına başarıyla yazıldığı anda sunucu tarafında CAPI ile `Lead` olayı fırlatın
   - Hemen sonrasında yanıtı istemciye döndürün

3. **Statik Başvuru ID'si Üretin**
   - Form gönderildiği an sunucuda üretilen `form_submission_id` oluşturun
   - Istemciye döndürüp Pixel `eventID` parametresine verin
   - Aynı ID'yi sunucudaki CAPI paketine `event_id` olarak ekleyin

4. **Spam & Bot Filtreleme**
   - ReCAPTCHA veya Cloudflare Turnstile doğrulamasını geçemeyen başvurular için Pixel veya CAPI tetiklemeyin
   - Doğrulama başarısız ise sessiz kalın

---

## 6. Standart ve Özel Etkinlik Parametre Yapısı {#6-standart-ve-özel-etkinlik-parametre-yapısı-tr}

### Standart Lead Etkinliği Parametre Şeması

```json
{
  "event_name": "Lead",
  "event_id": "lead_id_99812",
  "custom_data": {
    "lead_type": "Web Tasarım Teklif Formu",
    "value": 1500.00,
    "currency": "TRY"
  }
}
```

### Standart Etkinlikler Referansı

| Etkinlik | Açıklama | Kullanım Alanı | Zorunlu Parametreler |
|----------|----------|----------------|---------------------|
| `PageView` | Sayfa görüntüleme | Tüm sayfalar | - |
| `ViewContent` | Ürün detayı | Ürün sayfası | `content_ids`, `value` |
| `Search` | Site arama | Arama sonuçları | `search_string` |
| `AddToCart` | Sepete ekleme | Sepete ekle | `content_ids`, `value` |
| `InitiateCheckout` | Ödeme başlama | Ödeme sayfası | `content_ids`, `num_items` |
| `Purchase` | Satın alma | Sipariş onay | `content_ids`, `value`, `num_items` |
| `Lead` | Form başvurusu | İletişim formu | `value` (opsiyonel) |
| `CompleteRegistration` | Kayıt tamamlama | Kayıt sayfası | `status` |

---

## 7. Tam Kod Uygulamaları (JS, Node.js, Python, PHP) {#7-tam-kod-uygulamaları-js-nodejs-python-php-tr}

### 7.1. İstemci Tarafı (HTML/JS Base Code & Advanced Matching)

Tüm sayfaların `<head>` bölümüne ekleyin:

```html
<!-- Meta Pixel Base Code -->
<script>
!function(f,b,e,v,n,t,s){if(f.fbq)return;n=f.fbq=function(){n.callMethod?
n.callMethod.apply(n,arguments):n.queue.push(arguments)};
if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
n.queue=[];t=b.createElement(e);t.async=!0;
t.src=v;s=b.getElementsByTagName(e)[0];
s.parentNode.insertBefore(t,s)}(window, document,'script',
'https://connect.facebook.net/en_US/fbevents.js');

fbq('init', 'YOUR_DATASET_ID');
fbq('track', 'PageView');
</script>

<noscript>
  <img height="1" width="1" style="display:none"
       src="https://www.facebook.com/tr?id=YOUR_DATASET_ID&ev=PageView&noscript=1" />
</noscript>

<!-- Form Gönderimi Başarılı Olduğunda Tetiklenecek JS -->
<script>
function handleFormSuccess(responseData) {
  // responseData.submissionId sunucudan dönen benzersiz başvuru ID'sidir
  fbq('track', 'Lead', {
    content_name: 'İletişim Formu',
    value: 500.00,
    currency: 'TRY'
  }, {
    eventID: responseData.submissionId // Deduplication ID
  });
}
</script>
```

---

### 7.2. Node.js Sunucu Tarafı (CAPI + SHA-256 Hashing)

```javascript
const crypto = require('crypto');
const axios = require('axios');

// SHA-256 Hash Fonksiyonu
function hashData(data) {
  if (!data) return undefined;
  return crypto.createHash('sha256')
    .update(data.trim().toLowerCase())
    .digest('hex');
}

async function sendCapiLead({
  datasetId,
  accessToken,
  submissionId,
  eventSourceUrl,
  userData,
  customData
}) {
  
  const url = `https://graph.facebook.com/v25.0/${datasetId}/events`;

  const payload = {
    data: [{
      event_name: 'Lead',
      event_time: Math.floor(Date.now() / 1000),
      event_id: submissionId, // Deduplication ID
      event_source_url: eventSourceUrl,
      action_source: 'website',
      user_data: {
        em: userData.email ? [hashData(userData.email)] : undefined,
        ph: userData.phone ? [hashData(userData.phone)] : undefined,
        fn: userData.firstName ? [hashData(userData.firstName)] : undefined,
        ln: userData.lastName ? [hashData(userData.lastName)] : undefined,
        client_ip_address: userData.ipAddress,
        client_user_agent: userData.userAgent,
        fbp: userData.fbp,
        fbc: userData.fbc
      },
      custom_data: customData
    }],
    access_token: accessToken
  };

  try {
    const res = await axios.post(url, payload);
    console.log('✅ Lead CAPI başarıyla gönderildi:', res.data);
    return res.data;
  } catch (err) {
    console.error('❌ CAPI Hatası:', err.response ? err.response.data : err.message);
  }
}

// Kullanım Örneği
sendCapiLead({
  datasetId: 'YOUR_DATASET_ID',
  accessToken: 'YOUR_ACCESS_TOKEN',
  submissionId: 'lead_2026_8839',
  eventSourceUrl: 'https://example.com/iletisim',
  userData: {
    email: 'john@example.com',
    phone: '+905462388707',
    firstName: 'John',
    lastName: 'Doe',
    ipAddress: '185.190.10.1',
    userAgent: 'Mozilla/5.0...',
    fbp: 'fb.1.1711000000.123456789',
    fbc: 'fb.1.1711000000.IwAR0...'
  },
  customData: {
    value: 500.00,
    currency: 'TRY',
    lead_type: 'İletişim Formu'
  }
});
```

---

### 7.3. Python Sunucu Tarafı (CAPI + Deduplication)

```python
import time
import hashlib
import requests

def sha256_hash(val: str) -> str:
    if not val:
        return None
    return hashlib.sha256(
        val.strip().lower().encode('utf-8')
    ).hexdigest()

def send_capi_lead(dataset_id: str, access_token: str, submission_data: dict):
    endpoint = f"https://graph.facebook.com/v25.0/{dataset_id}/events"

    payload = {
        "data": [{
            "event_name": "Lead",
            "event_time": int(time.time()),
            "event_id": submission_data["submission_id"],  # Deduplication ID
            "event_source_url": submission_data["url"],
            "action_source": "website",
            "user_data": {
                "em": [sha256_hash(submission_data["email"])],
                "ph": [sha256_hash(submission_data["phone"])],
                "fn": [sha256_hash(submission_data["first_name"])],
                "ln": [sha256_hash(submission_data["last_name"])],
                "client_ip_address": submission_data["ip"],
                "client_user_agent": submission_data["user_agent"],
                "fbp": submission_data.get("fbp"),
                "fbc": submission_data.get("fbc")
            },
            "custom_data": {
                "value": 500.00,
                "currency": "TRY",
                "lead_type": submission_data.get("lead_type")
            }
        }],
        "access_token": access_token
    }

    res = requests.post(endpoint, json=payload)
    return res.json()

# Kullanım Örneği
if __name__ == "__main__":
    result = send_capi_lead(
        dataset_id="YOUR_DATASET_ID",
        access_token="YOUR_ACCESS_TOKEN",
        submission_data={
            "submission_id": "lead_2026_8839",
            "email": "john@example.com",
            "phone": "905462388707",
            "first_name": "John",
            "last_name": "Doe",
            "ip": "185.190.10.1",
            "user_agent": "Mozilla/5.0...",
            "url": "https://example.com/iletisim",
            "lead_type": "İletişim Formu"
        }
    )

    print("Meta API Yanıtı:", result)
```

---

### 7.4. PHP Form Başvuru (Lead CAPI Entegrasyonu)

```php
<?php

function send_meta_capi_lead($dataset_id, $access_token, $data) {
    $url = "https://graph.facebook.com/v25.0/{$dataset_id}/events";

    $user_data = [
        'em' => [hash('sha256', strtolower(trim($data['email'])))],
        'ph' => [hash('sha256', preg_replace('/[^0-9]/', '', $data['phone']))],
        'fn' => [hash('sha256', strtolower(trim($data['first_name'])))],
        'ln' => [hash('sha256', strtolower(trim($data['last_name'])))],
        'client_ip_address' => $_SERVER['REMOTE_ADDR'],
        'client_user_agent' => $_SERVER['HTTP_USER_AGENT'],
        'fbp' => $_COOKIE['_fbp'] ?? null,
        'fbc' => $_COOKIE['_fbc'] ?? null
    ];

    $payload = [
        'data' => [
            [
                'event_name' => 'Lead',
                'event_time' => time(),
                'event_id' => $data['submission_id'], // Deduplication ID
                'event_source_url' => $data['page_url'],
                'action_source' => 'website',
                'user_data' => $user_data,
                'custom_data' => [
                    'value' => 500.00,
                    'currency' => 'TRY',
                    'lead_type' => $data['lead_type'] ?? 'İletişim Formu'
                ]
            ]
        ],
        'access_token' => $access_token
    ];

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

    $response = curl_exec($ch);
    curl_close($ch);

    return json_decode($response, true);
}

// Kullanım Örneği
$result = send_meta_capi_lead(
    'YOUR_DATASET_ID',
    'YOUR_ACCESS_TOKEN',
    [
        'submission_id' => 'lead_2026_8839',
        'email' => 'john@example.com',
        'phone' => '905462388707',
        'first_name' => 'John',
        'last_name' => 'Doe',
        'page_url' => 'https://example.com/iletisim',
        'lead_type' => 'İletişim Formu'
    ]
);

echo json_encode($result);

?>
```

---

## 8. CMS ve E-Ticaret Panel Ayarları {#8-cms-ve-e-ticaret-panel-ayarları-tr}

### Shopify

- **Facebook & Instagram** kanalını yükleyin
- Data Sharing seviyesini **Maximum** seçin
- CAPI ve Gelişmiş Eşleştirme otomatik devreye girer
- Deduplication sistem tarafından yönetilir

### Özel Platformlar (İkas / Ticimax / Ideasoft)

1. Entegrasyonlar → **Meta Pixel & CAPI** menüsüne gidin
2. **Dataset ID** ve **Access Token** bilgisini girin
3. Otomatik sunucu tarafı CAPI modunu aktif edin

### WooCommerce

- **PixelYourSite Pro** eklentisini yükleyin
- CAPI Access Token ve Dataset ID girin
- **Automatic Advanced Matching** ve **Event ID Deduplication** seçeneklerini açın

---

## 9. Test, Denetim (Audit) ve Hata Ayıklama (Debugging) {#9-test-denetim-audit-ve-hata-ayıklama-debugging-tr}

### 1. Test Events (Test Etkinlikleri) Modu

1. Events Manager → **Test Events** sekmesine gidin
2. Ekranda görünen `TESTXXXXX` kodunu kopyalayın
3. CAPI isteğinize `"test_event_code": "TESTXXXXX"` parametresini ekleyin
4. Formu gönderin ve Events Manager'da doğrulayın:
   - **Browser** ve **Server** sinyalleri aynı `event_id` ile eşleşmiş olmalı
   - **"Deduplicated"** etiketi görülmeli

### 2. Tarayıcı Test Araçları

- **Meta Pixel Helper** Chrome Uzantısı
  - İstemci tarafı Pixel'in doğru çalışıp çalışmadığını kontrol edin
  - Parametre hatalarını belirleyin
  - Network isteklerini izleyin

### 3. Hedef Performans Metrikleri Kontrol Listesi

- ✅ **Event Coverage Ratio**: CAPI/Pixel etkinlik oranı **%75 ve üzeri**
- ✅ **EMQ Skoru**: Lead ve Purchase olaylarında **8.0 ve üzeri**
- ✅ **1:1 Eşleşme**: Günlük veritabanı kayıtları vs Meta raporları farkı **%0-2 bandında**

---

## 10. KVKK, GDPR ve Gizlilik Uyumluluğu {#10-kvkk-gdpr-ve-gizlilik-uyumluluğu-tr}

### Kullanıcı Rıza Yönetimi

- Kullanıcı takip izni vermediğinde: `fbq('consent', 'revoke')` çağırın
- Sunucu tarafında: Rıza olmadığında CAPI etkinliklerini göndermeyin
- Do Not Track (DNT) tarayıcı ayarlarına saygı gösterin

### SHA-256 Şifreleme Zorunludur

- **Asla ham PII göndermeyiniz** (e-posta, telefon, ad-soyad)
- Her zaman iletimden önce SHA-256 ile hash edin
- API çağrılarından önce yerel olarak hash edin

### Veri İşleme Seçenekleri (AB Yönetmelikleri)

- California CCPA: `data_processing_options` parametresini ayarlayın
- AB GDPR: Veri işleme konumunu yapılandırın
- Meta, veri saklama ve silme seçenekleri sağlar

---

## Kaynaklar & Bağlantılar / Resources & Links

- [Meta for Developers - Conversions API](https://developers.facebook.com/documentation/ads-commerce/conversions-api)
- [Meta Business Help - EMQ Guide](https://www.facebook.com/business/help/765081237991954)
- [Cuma Karadaş GitHub](https://github.com/cumakaradash)
- [Meta Events Manager](https://business.facebook.com/)

---

**Last Updated / Son Güncelleme:** August 2026 / Ağustos 2026  
**Version / Versiyon:** 2.0  
**License / Lisans:** MIT

---

> ** How to Use This Document / Bu Dokümantı Nasıl Kullanılır:**
> - Use the Quick Navigation links at the top to jump to your preferred language / Tercih ettiğiniz dile gitmek için yukarıdaki Hızlı Navigasyon bağlantılarını kullanın
> - Each section has cross-references marked with [Link] for easy navigation / Her bölümde kolay navigasyon için işaretlenmiş çapraz referanslar bulunur
> - For questions, refer to the official Meta Developer documentation / Sorularınız için resmi Meta Developer belgelendirmesine başvurun

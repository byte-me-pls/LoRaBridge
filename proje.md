# LoRa-to-Modbus TCP Secure Gateway

## 📝 Proje Hakkında

Bu proje, sahadaki uç birimlerden (node) gelen kritik sensör verilerini **LoRa** teknolojisi ile uzun menzilli (5 km+) ve düşük güç tüketimiyle toplayıp, endüstriyel standart olan **Modbus TCP** protokolüne dönüştüren bir geçit yolu (Gateway) tasarımıdır.

Proje özellikle kablo çekilmesinin zor olduğu **savunma sanayii test alanları**, geniş **tarım arazileri** ve **sanayi tesisleri**nde, sahadaki verilerin merkezi SCADA/HMI sistemlerine güvenli ve kablosuz entegrasyonunu hedefler.

### Motivasyon

| Problem | Geleneksel Çözüm | Bu Projenin Çözümü |
|---------|-------------------|---------------------|
| Uzun mesafe veri iletimi | Kablolu RS-485 / Ethernet | LoRa RF (5 km+ menzil) |
| Enerji tüketimi | Sürekli beslemeli cihazlar | LoRa düşük güç (<50 mA TX) |
| Endüstriyel entegrasyon | Özel protokoller | Standart Modbus TCP |
| Veri güvenliği | Açık RF kanallar | AES-128 şifrelenmiş paketler |
| Çoklu erişim | Tek istemci | Multi-client TCP bağlantı |

---

## 🏆 Farkımız — Neden Bu Proje?

Piyasada LoRa gateway çözümleri mevcut ancak bu proje aşağıdaki kritik noktalarda ayrışır:

### Rakip Karşılaştırması

| Özellik | Ticari LoRa Gateway'ler | LoRaWAN (TTN/ChirpStack) | **Bu Proje** |
|---------|------------------------|--------------------------|--------------|
| Maliyet | $300–$1000+ | Ücretsiz (altyapı maliyeti var) | **~$15** (ESP32 + SX1278) |
| Modbus TCP çıkışı | Çoğunda yok, ek dönüştürücü gerekir | ❌ Yok (MQTT/HTTP çıkışı) | ✅ **Doğrudan Modbus TCP** |
| Uçtan uca şifreleme | Çoğunda opsiyonel / ücretli | Var (AppSKey) ama altyapı gerektirir | ✅ **AES-128 dahili, sıfır altyapı** |
| Bulut bağımlılığı | Bazıları bulut zorunlu | TTN sunucusu veya self-hosted | ✅ **Bulut YOK, tamamen lokal** |
| SCADA entegrasyonu | Genelde ek yazılım/lisans gerekir | Ek Modbus bridge gerekir | ✅ **Doğrudan SCADA'ya bağlanır** |
| Çakışma yönetimi | TDMA (pahalı) veya yok | ALOHA (çakışma riski yüksek) | ✅ **Dönen Lider Konsensüs** |
| Kaynak kodu | Kapalı kaynak | Açık (sunucu tarafı) | ✅ **Tamamen açık kaynak** |
| Kurulum karmaşıklığı | Orta–yüksek | Yüksek (Network/App Server) | ✅ **Tak-çalıştır basitliği** |

### 6 Temel Fark

**1. 🔐 Dahili Kriptografik Katman**
Piyasadaki çoğu düşük maliyetli gateway şifreleme sunmaz. Bu projede AES-128-CBC + HMAC-SHA256 **firmware seviyesinde gömülüdür** — ek donanım veya yazılım gerektirmez. Savunma sanayii gereksinimleri için kritiktir.

**2. 🚫 LoRaWAN'a Bağımlılık Yok**
LoRaWAN kullanmak için TTN sunucusu, ChirpStack kurulumu veya bulut aboneliği gerekir. Bu proje **saf LoRa (point-to-star)** kullanır — Gateway ve Node'lar arasında doğrudan iletişim kurar. Sıfır altyapı maliyeti.

**3. 🔄 Dönen Lider Konsensüs Mekanizması**
LoRaWAN'ın ALOHA protokolü yoğun node ortamında %30+ paket kaybına yol açar. Bu projede **zaman dilimi tabanlı dönen lider** mekanizması ile çakışma **sıfıra** indirilir — pahalı TDMA donanımı gerektirmeden.

**4. 🏭 Doğrudan SCADA/PLC Entegrasyonu**
Rakip çözümler veriyi MQTT veya HTTP ile buluta gönderir, sonra ek bir bridge ile Modbus'a dönüştürür. Bu projede **SCADA, Gateway'e doğrudan Modbus TCP ile bağlanır** — ek yazılım, lisans veya internet bağlantısı gerekmez.

**5. 💰 Ultra Düşük Maliyet**
Ticari LoRa-Modbus gateway'ler $300–$1000 arasıdır. Bu projenin toplam donanım maliyeti:

| Bileşen | Birim Fiyat | Adet | Toplam |
|---------|-------------|------|--------|
| ESP32-WROOM-32 | ~$5 | 1 (Gateway) | $5 |
| SX1278 LoRa Modül | ~$4 | 1 | $4 |
| Anten (868 MHz) | ~$2 | 1 | $2 |
| Diğer (kablo, breadboard) | ~$4 | 1 | $4 |
| **Gateway Toplam** | | | **~$15** |

**6. 📖 Tamamen Açık Kaynak**
Tüm firmware, dokümantasyon ve simülasyon araçları açık kaynak olarak sunulur. Ticari çözümlerin kapalı kaynak doğası, özelleştirme ve denetim imkânını kısıtlar.

---

## 🏗 Sistem Mimarisi

### Genel Bakış

```
┌──────────────┐     LoRa RF (868 MHz)     ┌──────────────────┐    TCP/IP (Wi-Fi)    ┌──────────────┐
│  Saha Node 1 │ ─────────────────────────► │                  │ ◄─────────────────── │   SCADA/HMI  │
│  (ESP32 +    │     5 km+ menzil           │   GATEWAY        │    Modbus TCP        │   İstemci    │
│   SX1278 +   │                            │   (ESP32 +       │    Port 502          └──────────────┘
│   Sensörler) │                            │    SX1278 +      │
└──────────────┘                            │    Wi-Fi)        │ ◄─────────────────── ┌──────────────┐
                                            │                  │    Modbus TCP        │  Mobil App   │
┌──────────────┐     LoRa RF (şifreli)      │   ┌──────────┐  │                      └──────────────┘
│  Saha Node 2 │ ─────────────────────────► │   │ AES-128  │  │
└──────────────┘                            │   │ Şifre    │  │ ◄─────────────────── ┌──────────────┐
                                            │   │ Çözme    │  │    Modbus TCP        │     PLC      │
┌──────────────┐     LoRa RF (şifreli)      │   └──────────┘  │                      └──────────────┘
│  Saha Node N │ ─────────────────────────► │                  │
└──────────────┘                            └──────────────────┘
                                                    │
                                             ┌──────┴──────┐
                                             │ Web Panel   │
                                             │ Port 80     │
                                             │ (Debug)     │
                                             └─────────────┘
```

### Katmanlı Mimari

| Katman | Protokol | Açıklama |
|--------|----------|----------|
| **Fiziksel** | LoRa (SX1278) | 868 MHz ISM bandı, SF7–SF12, BW 125 kHz |
| **Güvenlik** | AES-128-CBC + HMAC | Veri şifreleme ve bütünlük doğrulama |
| **Uygulama** | Özel paket formatı | Node ID + Sensör verisi + CRC |
| **Dönüşüm** | Gateway iç mantık | LoRa paketi → Modbus Register eşlemesi |
| **Endüstriyel** | Modbus TCP | Holding Register (FC 0x03), Port 502 |
| **Yönetim** | HTTP | Web tabanlı izleme paneli, Port 80 |

---

## 🛠 Donanım Bileşenleri

### Gateway Donanımı

| Bileşen | Model | Görev |
|---------|-------|-------|
| Mikrodenetleyici | ESP32-WROOM-32 | Ana işlemci, Wi-Fi, TCP/IP yığını |
| LoRa Modülü | Semtech SX1278 | 868 MHz RF alıcı/verici |
| Anten | SMA 868 MHz | LoRa sinyal alımı |
| Güç Kaynağı | 5V / 2A USB veya 3.3V LDO | Sistem beslemesi |

### ESP32 ↔ SX1278 Pin Bağlantıları

| SX1278 Pin | ESP32 GPIO | Açıklama |
|------------|------------|----------|
| SCK | GPIO 18 | SPI Clock |
| MISO | GPIO 19 | SPI Master In |
| MOSI | GPIO 23 | SPI Master Out |
| NSS (CS) | GPIO 5 | SPI Chip Select |
| RST | GPIO 14 | Modül Reset |
| DIO0 | GPIO 2 | Interrupt (RX Done) |
| 3.3V | 3V3 | Güç beslemesi |
| GND | GND | Topraklama |

### Saha Node Donanımı

| Bileşen | Model | Görev |
|---------|-------|-------|
| Mikrodenetleyici | ESP32-WROOM-32 | Sensör okuma, LoRa iletim |
| LoRa Modülü | Semtech SX1278 | 868 MHz RF verici |
| Sıcaklık Sensörü | DHT22 / DS18B20 | Ortam sıcaklığı ölçümü |
| Nem Sensörü | DHT22 | Bağıl nem ölçümü |
| Basınç Sensörü | BMP280 | Atmosfer basıncı ölçümü |
| Batarya | Li-Ion 3.7V 2000mAh | Mobil güç kaynağı |

---

## 📡 LoRa Haberleşme Protokolü

### Paket Formatı

```
┌──────────┬──────────┬───────────┬────────────┬──────────┬──────┐
│ Preamble │ Node ID  │ Msg Type  │  Payload   │   HMAC   │ CRC  │
│ (8 byte) │ (2 byte) │ (1 byte)  │ (N byte)   │ (4 byte) │(2 b) │
└──────────┴──────────┴───────────┴────────────┴──────────┴──────┘
                                   │
                                   ▼  (AES-128-CBC ile şifrelenmiş)
                       ┌────────────────────────────────┐
                       │ Sensor Count │ Sensor 1 Data   │
                       │   (1 byte)   │  ID (1) + Val(4)│
                       │              │ Sensor 2 Data   │
                       │              │  ID (1) + Val(4)│
                       │              │      ...        │
                       └────────────────────────────────┘
```

### Mesaj Tipleri

| Tip Kodu | Açıklama | Yön |
|----------|----------|-----|
| `0x01` | Sensör veri paketi | Node → Gateway |
| `0x02` | Durum bildirimi (heartbeat) | Node → Gateway |
| `0x03` | Yapılandırma komutu | Gateway → Node |
| `0x04` | ACK (onay) | Gateway → Node |
| `0xFF` | Hata bildirimi | Çift yönlü |

### LoRa Parametreleri

| Parametre | Değer | Açıklama |
|-----------|-------|----------|
| Frekans | 868 MHz | Avrupa ISM bandı (Türkiye uyumlu) |
| Yayılma Faktörü (SF) | SF7 – SF12 | Menzil/hız dengesi (varsayılan SF10) |
| Bant Genişliği (BW) | 125 kHz | Standart LoRa BW |
| Kodlama Oranı (CR) | 4/5 | Hata düzeltme oranı |
| TX Gücü | 14 dBm (25 mW) | Maksimum izin verilen güç |
| Teorik Menzil | 5–15 km | Görüş hattına bağlı |

---

## 🔐 Güvenlik Katmanı

### Kriptografik Tasarım

Tüm LoRa paketleri **AES-128-CBC** ile şifrelenir ve **HMAC-SHA256** ile bütünlük kontrolü yapılır.

```
┌─────────────────────────────────────────────────────┐
│                   Şifreleme Akışı                    │
│                                                      │
│  Ham Veri ──► AES-128-CBC Şifreleme ──► Şifreli Veri │
│                    │                                  │
│              Önceden Paylaşılmış                      │
│              Anahtar (PSK)                            │
│                                                      │
│  Şifreli Veri ──► HMAC-SHA256 ──► Bütünlük Etiketi   │
│                                                      │
│  [Şifreli Veri + HMAC] ──► LoRa ile Gönderim         │
└─────────────────────────────────────────────────────┘
```

### Güvenlik Özellikleri

| Özellik | Yöntem | Amaç |
|---------|--------|------|
| Gizlilik | AES-128-CBC | Veri içeriğinin 3. şahıslardan gizlenmesi |
| Bütünlük | HMAC-SHA256 | Verinin iletim sırasında değiştirilmediğinin doğrulanması |
| Anahtar Yönetimi | Statik PSK | Önceden paylaşılmış simetrik anahtar |
| Tekrar Saldırısı Koruması | Paket sıra numarası | Aynı paketin tekrar gönderilmesinin engellenmesi |

---

## 📊 Modbus TCP Register Haritası

Gateway, sahadaki sensör verilerini aşağıdaki Modbus Holding Register adreslerine eşler. SCADA/HMI sistemleri bu adresleri **Function Code 0x03 (Read Holding Registers)** ile okur.

### Node 1 — Register Adresleri (0–9)

| Register Adresi | Veri Tipi | Ölçek | Birim | Açıklama |
|-----------------|-----------|-------|-------|----------|
| 0 | UINT16 | 1 | — | Node 1 Durum (0=Offline, 1=Online) |
| 1 | INT16 | ×0.1 | °C | Node 1 Sıcaklık |
| 2 | UINT16 | ×0.1 | %RH | Node 1 Nem |
| 3 | UINT16 | ×0.1 | hPa | Node 1 Basınç |
| 4 | INT16 | 1 | dBm | Node 1 RSSI (sinyal gücü) |
| 5 | UINT16 | 1 | — | Node 1 Batarya (%) |
| 6 | UINT16 | 1 | sn | Node 1 Son Görülme (saniye önce) |
| 7–9 | — | — | — | Rezerve |

### Node 2–10 — Register Adresleri (10–99)

Her node için 10 register ayrılmıştır. Node N'in başlangıç adresi: `(N-1) × 10`

| Node | Register Aralığı |
|------|------------------|
| Node 1 | 0 – 9 |
| Node 2 | 10 – 19 |
| Node 3 | 20 – 29 |
| ... | ... |
| Node 10 | 90 – 99 |

### Sistem Register Adresleri (100–109)

| Register Adresi | Açıklama |
|-----------------|----------|
| 100 | Toplam aktif node sayısı |
| 101 | Gateway uptime (dakika) |
| 102 | Toplam alınan paket sayısı |
| 103 | Hatalı paket sayısı (CRC hatası) |
| 104 | Şifre çözme hatası sayısı |
| 105 | Aktif Modbus TCP bağlantı sayısı |
| 106–109 | Rezerve |

---

## 🔄 Dönen Lider Konsensüs Mekanizması

Birden fazla saha node'unun aynı LoRa kanalını paylaştığı ortamda, **çakışmaları önlemek** ve **bant genişliğini adil dağıtmak** için Dönen Lider (Rotating Leader) konsensüs mekanizması kullanılır.

### Çalışma Prensibi

```
Zaman ──────────────────────────────────────────────────►

 Dilim 1      Dilim 2      Dilim 3      Dilim 4      Dilim 5
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Node 1   │ Node 2   │ Node 3   │ Node 1   │ Node 2   │
│ (LİDER)  │ (LİDER)  │ (LİDER)  │ (LİDER)  │ (LİDER)  │
│ → Gönder │ → Gönder │ → Gönder │ → Gönder │ → Gönder │
└──────────┴──────────┴──────────┴──────────┴──────────┘
   10 sn       10 sn       10 sn       10 sn       10 sn
```

### Kurallar

1. Her node'a benzersiz bir **sıra numarası** (1, 2, 3, ...) atanır
2. Her **zaman dilimi** (varsayılan 10 saniye) boyunca yalnızca lider node veri gönderebilir
3. Liderlik sırayla bir sonraki node'a geçer: `lider = (zaman / dilim_süresi) % node_sayısı + 1`
4. Node, kendi dilimi dışında **sessiz kalır** (RF yayını yapmaz)
5. Bir node iletişim kaybederse, dilimi atlanır ve sıradaki node devam eder

### Avantajlar

| Özellik | Açıklama |
|---------|----------|
| Çakışma önleme | Aynı anda yalnızca bir node yayın yapar |
| Adil paylaşım | Her node eşit gönderim süresi alır |
| Düşük karmaşıklık | Merkezi koordinasyon gerektirmez (zaman tabanlı) |
| Ölçeklenebilirlik | Yeni node eklemek için sadece sıra numarası ataması yeterli |

---

## 🖥 Web Hata Ayıklama Paneli

Gateway üzerinde çalışan **ESPAsyncWebServer** tabanlı web arayüzü, canlı izleme ve hata ayıklama olanağı sunar.

### Erişim

- **URL:** `http://<gateway_ip>:80`
- **Özellikler:**
  - ✅ Canlı register değerleri tablosu (otomatik yenileme)
  - ✅ Son 50 LoRa paketin logu (zaman damgası, node ID, RSSI, veri)
  - ✅ Sistem durumu (uptime, bellek kullanımı, Wi-Fi sinyal gücü)
  - ✅ Aktif Modbus TCP bağlantı listesi
  - ✅ Node sağlık durumu haritası

---

## 📂 Proje Yapısı

```
Modbus/
├── src/                      # ESP32 kaynak kodları (C++ / Arduino)
│   ├── main.cpp              # Ana program (setup + loop)
│   ├── config.h              # Yapılandırma sabitleri
│   ├── lora_handler.h/.cpp   # LoRa SX1278 sürücü katmanı
│   ├── modbus_server.h/.cpp  # Modbus TCP sunucu
│   ├── crypto_layer.h/.cpp   # AES-128 şifreleme katmanı
│   ├── register_map.h        # Register adres tanımları
│   ├── web_dashboard.h/.cpp  # HTTP debug paneli
│   ├── consensus.h/.cpp      # Dönen lider mekanizması
│   └── node_sensor.cpp       # Saha node firmware (ayrı cihaz)
│
├── docs/                     # Teknik dokümantasyon
│   ├── register_map.md       # Modbus register detaylı listesi
│   ├── hardware_setup.md     # Devre şeması ve bağlantılar
│   ├── protocol_flow.md      # Veri akış diyagramı
│   └── security_spec.md      # Güvenlik katmanı spesifikasyonu
│
├── simulation/               # Ağ modelleme ve simülasyon (C# / ASP.NET Web)
│   ├── Program.cs            # Local web server başlatma
│   ├── NetworkTopology.cs    # Ağ topolojisi görselleştirme
│   ├── ConsensusSim.cs       # Konsensüs mekanizması simülasyonu
│   └── ModbusTestClient.cs   # Modbus TCP test istemcisi
│
├── platformio.ini            # PlatformIO proje konfigürasyonu
├── proje.md                  # Proje açıklama dökümanı (bu dosya)
└── README.md                 # GitHub README (İngilizce)
```

---

## 📅 Proje Zaman Çizelgesi

| Hafta | Görev | Çıktı |
|-------|-------|-------|
| 1–2 | Donanım temin ve devre kurulumu | Çalışan ESP32 + SX1278 devresi |
| 3–4 | LoRa haberleşme ve paket formatı | İki ESP32 arası veri iletimi |
| 5–6 | AES şifreleme katmanı entegrasyonu | Şifreli LoRa veri transferi |
| 7–8 | Modbus TCP sunucu geliştirme | SCADA'dan register okuma testi |
| 9–10 | Web paneli ve sistem entegrasyonu | Tam çalışan gateway prototipi |
| 11–12 | Test, dokümantasyon ve sunum hazırlığı | Final raporu ve demo |

---

## 🔧 Kurulum Senaryosu

Sistemin sıfırdan kurulumu aşağıdaki adımlarla gerçekleştirilir:

### Adım 1 — Donanım Montajı

```
1. ESP32 Gateway kartına SX1278 LoRa modülünü bağlayın (pin tablosuna bakınız)
2. 868 MHz SMA anteni SX1278'e takın
3. ESP32'yi USB ile bilgisayara bağlayın
4. Saha node'ları için aynı işlemi tekrarlayın (ESP32 + SX1278 + sensörler)
```

### Adım 2 — Firmware Yükleme

```
1. PlatformIO IDE'yi kurun (VS Code eklentisi)
2. Proje klasörünü açın
3. src/config.h dosyasından ayarları yapılandırın:
   - WIFI_SSID / WIFI_PASS    → Gateway'in bağlanacağı Wi-Fi ağı
   - LORA_FREQUENCY            → 868E6 (868 MHz)
   - LORA_SPREADING_FACTOR     → 10 (SF10, varsayılan)
   - AES_KEY                   → 16 byte şifreleme anahtarı
   - MODBUS_PORT               → 502
   - MAX_NODES                 → 10
4. Gateway firmware'ini yükleyin:
   > pio run --target upload --environment gateway
5. Node firmware'ini yükleyin (her node için):
   > pio run --target upload --environment node
```

### Adım 3 — Ağ Yapılandırması

```
1. Gateway ESP32 açıldığında Wi-Fi ağına otomatik bağlanır
2. Seri monitörden Gateway IP adresini okuyun:
   > [BOOT] Wi-Fi bağlandı. IP: 192.168.1.50
3. Web hata ayıklama panelini açın:
   > Tarayıcı → http://192.168.1.50
4. Node'ların bağlantı durumunu panelden kontrol edin
```

### Adım 4 — SCADA / HMI Bağlantısı

```
1. SCADA yazılımında yeni bir Modbus TCP cihazı tanımlayın:
   - IP Adresi  : 192.168.1.50  (Gateway IP)
   - Port       : 502
   - Unit ID    : 1
   - Fonksiyon  : 0x03 (Read Holding Registers)
2. Register adreslerini eşleyin (register haritasına bakınız):
   - Register 1 → Node 1 Sıcaklık (×0.1 °C)
   - Register 2 → Node 1 Nem (×0.1 %RH)
   - ...
3. Polling süresini ayarlayın (önerilen: 5 saniye)
4. SCADA ekranında sensör verilerini canlı izlemeye başlayın
```

### Adım 5 — Simülasyon ve Test (Opsiyonel)

```
1. .NET 8.0 SDK'yı kurun
2. Simülasyon klasörüne gidin ve web sunucuyu başlatın:
   > cd simulation
   > dotnet run
3. Tarayıcıda açın:
   > http://localhost:5000
4. Modbus TCP test istemcisi ile Gateway'e bağlanın
5. Ağ topolojisi ve konsensüs simülasyonunu çalıştırın
```

---

## 📋 Kullanım Senaryoları

### Senaryo 1 — Savunma Sanayii Test Alanı

```
📍 Konum: Açık arazi test sahası (2 km²)
🎯 Amaç: Silah sistemi test atışları sırasında çevresel verilerin izlenmesi
```

| Aşama | Açıklama |
|-------|----------|
| **Kurulum** | 5 adet saha node'u test alanına yerleştirilir (sıcaklık, titreşim, basınç sensörleri) |
| **Veri Toplama** | Node'lar her 10 saniyede LoRa ile AES şifreli veri gönderir |
| **Dönüştürme** | Gateway verileri Modbus TCP register'larına yazar |
| **İzleme** | Komuta merkezi SCADA ekranından tüm node'ların anlık verilerini izler |
| **Güvenlik** | AES-128 şifreleme ile RF dinleme saldırılarına karşı koruma sağlanır |
| **Avantaj** | Kablo çekimi gerektirmez, node'lar batarya ile çalışır, hızlı kurulum/söküm |

### Senaryo 2 — Akıllı Tarım (Sera / Tarla)

```
📍 Konum: 50 dönümlük tarım arazisi
🎯 Amaç: Toprak nemi, hava sıcaklığı ve rüzgar hızının merkezi izlenmesi
```

| Aşama | Açıklama |
|-------|----------|
| **Kurulum** | 8 node arazi boyunca stratejik noktalara yerleştirilir |
| **Veri Toplama** | Her node toprak nemi (kapasitif sensör), sıcaklık ve nem verisini gönderir |
| **Dönüştürme** | Gateway verileri Wi-Fi üzerinden çiftlik binasındaki sunucuya aktarır |
| **İzleme** | Çiftçi, mobil uygulama veya bilgisayardan Modbus TCP ile verileri okur |
| **Karar Destek** | Belirli register değerleri eşik aşarsa sulama veya havalandırma tetiklenir |
| **Avantaj** | Geniş arazide kablosuz kapsama, düşük enerji tüketimi, uzun batarya ömrü |

### Senaryo 3 — Endüstriyel Tesis İzleme

```
📍 Konum: Fabrika sahası (birden fazla bina)
🎯 Amaç: Uzak üretim birimlerindeki makine sıcaklıklarının merkezi SCADA'ya aktarımı
```

| Aşama | Açıklama |
|-------|----------|
| **Kurulum** | Her üretim biriminde 1–2 node, motor/kompresör sıcaklıkları ölçer |
| **Veri Toplama** | Node'lar PT100/termoçift sensörlerinden okunan verileri LoRa ile iletir |
| **Dönüştürme** | Merkezi kontrol odasındaki Gateway, verileri mevcut SCADA'ya entegre eder |
| **İzleme** | Operatör mevcut HMI ekranlarından uzak birimlerin verilerini görür |
| **Alarm** | PLC, Modbus register değerlerinden aşırı sıcaklık alarmı üretir |
| **Avantaj** | Mevcut SCADA altyapısına ek kablo çekmeden kablosuz genişleme |

---

## 🛡 Kullanılan Teknolojiler

| Teknoloji | Versiyon | Kullanım Alanı |
|-----------|----------|----------------|
| ESP32-WROOM-32 | — | Mikrodenetleyici (Gateway + Node) |
| Semtech SX1278 | — | LoRa RF modülü |
| Arduino Framework | 2.x | ESP32 firmware geliştirme |
| PlatformIO | 6.x | Derleme ve proje yönetimi |
| LoRa Kütüphanesi | sandeepmistry/LoRa | SX1278 sürücüsü |
| eModbus | eModbus/eModbus | Modbus TCP sunucu |
| ESPAsyncWebServer | me-no-dev | HTTP debug paneli |
| ArduinoJson | 7.x | JSON veri işleme |
| AES (mbedTLS) | — | Kriptografik işlemler |
| C# / ASP.NET | .NET 8.0+ | Lokal web sunucu, simülasyon ve test araçları |

---

## 📧 İletişim

**Muhammet Emin Yıldız**
Gazi Üniversitesi — TUSAŞ Kontrol ve Otomasyon Mühendisliği
📩 [meyzdev@gmail.com](mailto:meyzdev@gmail.com)
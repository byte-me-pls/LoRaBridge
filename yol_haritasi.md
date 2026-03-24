# 🗺 LoRa-to-Modbus TCP Gateway — Uygulama Yol Haritası

Bu döküman, projeyi sıfırdan hayata geçirmek için gereken tüm aşamaları, alt görevleri ve çıktıları detaylandırır.

---

## Faz 0 — Hazırlık ve Tedarik (1 Hafta)

### 0.1 Donanım Tedarik Listesi

| # | Bileşen | Adet | Tahmini Fiyat | Nereden? |
|---|---------|------|---------------|----------|
| 1 | ESP32-WROOM-32 DevKit | 3 (1 gateway + 2 node) | ~₺250/adet | Robotistan, Direnc.net, AliExpress |
| 2 | SX1278 LoRa Modül (868 MHz) | 3 | ~₺120/adet | AliExpress, Robotistan |
| 3 | 868 MHz SMA Anten | 3 | ~₺30/adet | AliExpress |
| 4 | DHT22 Sıcaklık/Nem Sensörü | 2 | ~₺50/adet | Robotistan |
| 5 | BMP280 Basınç Sensörü | 2 | ~₺40/adet | Robotistan |
| 6 | Breadboard + Jumper Kablo Seti | 1 set | ~₺60 | Robotistan |
| 7 | USB Kablo (Micro-USB veya Type-C) | 3 | ~₺20/adet | Mevcut olabilir |
| **Toplam** | | | **~₺1.100–1.400** | |

### 0.2 Yazılım Kurulumu

- [ ] VS Code kur
- [ ] PlatformIO IDE eklentisini kur (VS Code Extensions)
- [ ] Git kur ve GitHub repo oluştur
- [ ] .NET 8.0 SDK kur (simülasyon için)
- [ ] Seri port monitör aracı kur (PlatformIO dahili veya PuTTY)
- [ ] Modbus TCP test yazılımı kur (ModRSsim2 veya QModMaster — ücretsiz)

### 0.3 Proje İskeleti

- [ ] GitHub repo'da klasör yapısını oluştur (`src/`, `docs/`, `simulation/`)
- [ ] `platformio.ini` dosyasını oluştur (ESP32 board + kütüphane tanımları)
- [ ] Boş `config.h` şablonu oluştur
- [ ] İlk commit: "Initial project structure"

**✅ Faz 0 Çıktısı:** Tüm donanım elde, geliştirme ortamı hazır, GitHub repo canlı.

---

## Faz 1 — LoRa Haberleşme (2 Hafta)

### 1.1 Donanım Bağlantısı (2 gün)

- [ ] ESP32 #1 (Gateway) + SX1278 bağlantısını yap (pin tablosuna göre)
- [ ] ESP32 #2 (Node) + SX1278 bağlantısını yap
- [ ] Bağlantıları multimetre ile doğrula (3.3V, GND, SPI sinyalleri)
- [ ] SX1278 modülün hayatta olduğunu kontrol et (SPI register okuma testi)

### 1.2 Basit LoRa Gönder/Al Testi (3 gün)

- [ ] `sandeepmistry/LoRa` kütüphanesini `platformio.ini`'ye ekle
- [ ] Node tarafı: her 5 saniyede "Hello" mesajı gönder
- [ ] Gateway tarafı: gelen mesajı seri porta yazdır
- [ ] RSSI ve SNR değerlerini logla
- [ ] **Test:** İki cihaz yan yana çalışıyor mu? Mesaj ulaşıyor mu?

### 1.3 Yapılandırılmış Paket Formatı (3 gün)

- [ ] `lora_handler.h` / `lora_handler.cpp` oluştur
- [ ] Paket yapısını tanımla: `[Node ID (2B) | Msg Type (1B) | Payload (NB) | CRC (2B)]`
- [ ] Node tarafı: sensör verisini paketleyip gönder
- [ ] Gateway tarafı: paketi ayrıştır, Node ID ve sensör değerlerini çıkar
- [ ] CRC kontrolü ekle (bozuk paketleri reddet)
- [ ] **Test:** Gateway seri monitörde "Node 1: Temp=25.3°C, Hum=60.2%" yazıyor mu?

### 1.4 Çoklu Node Desteği (2 gün)

- [ ] İkinci node'u bağla (ESP32 #3)
- [ ] Her node'a farklı ID ata (config.h'den)
- [ ] Gateway'de node tablosu oluştur (hangi node ne zaman son veri gönderdi?)
- [ ] **Test:** İki node sırayla veri gönderiyor, gateway ikisini de ayrı ayrı tanıyor mu?

**✅ Faz 1 Çıktısı:** 2 node → 1 gateway, yapılandırılmış paketler, CRC doğrulama, çoklu node takibi.

---

## Faz 2 — Güvenlik Katmanı (1.5 Hafta)

### 2.1 AES-128 Şifreleme (3 gün)

- [ ] `crypto_layer.h` / `crypto_layer.cpp` oluştur
- [ ] ESP32'nin dahili mbedTLS kütüphanesini kullan (ek kütüphane gereksiz)
- [ ] AES-128-CBC modunda şifreleme fonksiyonu yaz
- [ ] AES-128-CBC modunda şifre çözme fonksiyonu yaz
- [ ] `config.h`'de 16 byte PSK (Pre-Shared Key) tanımla
- [ ] **Test:** Aynı anahtarla şifrelenen veri, diğer tarafta doğru çözülüyor mu?

### 2.2 HMAC Bütünlük Kontrolü (2 gün)

- [ ] HMAC-SHA256 hesaplama fonksiyonu yaz
- [ ] Gönderici: [şifreli veri + HMAC] olarak paketle
- [ ] Alıcı: önce HMAC doğrula, sonra şifre çöz
- [ ] Yanlış anahtarla gelen paketi reddet
- [ ] **Test:** Paketi el ile boz → gateway reddediyor mu?

### 2.3 Tekrar Saldırısı Koruması (2 gün)

- [ ] Her pakete artan sıra numarası ekle
- [ ] Gateway'de her node için son sıra numarasını tut
- [ ] Eski veya aynı sıra numaralı paketleri reddet
- [ ] **Test:** Aynı paketi iki kez gönder → ikincisi reddediliyor mu?

### 2.4 Entegrasyon (1 gün)

- [ ] Faz 1'deki LoRa iletişimini şifreli hale getir
- [ ] Node: sensör verisi → AES şifrele → HMAC ekle → LoRa gönder
- [ ] Gateway: LoRa al → HMAC doğrula → AES çöz → veri oku
- [ ] **Test:** Şifreli iletişim Faz 1 kadar stabil çalışıyor mu?

**✅ Faz 2 Çıktısı:** Uçtan uca şifreli LoRa iletişimi, bütünlük kontrolü, tekrar saldırısı koruması.

---

## Faz 3 — Modbus TCP Sunucu (1.5 Hafta)

### 3.1 Wi-Fi Bağlantısı (1 gün)

- [ ] Gateway ESP32'yi Wi-Fi'ye bağla (config.h'den SSID/şifre)
- [ ] IP adresini seri porta yazdır
- [ ] Bağlantı kesilirse otomatik yeniden bağlan
- [ ] **Test:** Gateway her açıldığında Wi-Fi'ye bağlanıyor mu?

### 3.2 Modbus TCP Sunucu Kurulumu (3 gün)

- [ ] `eModbus/eModbus` kütüphanesini `platformio.ini`'ye ekle
- [ ] `modbus_server.h` / `modbus_server.cpp` oluştur
- [ ] Port 502'de Modbus TCP sunucu başlat
- [ ] `register_map.h` tanımla (Node 1: reg 0–9, Node 2: reg 10–19, ...)
- [ ] Holding Register'ları oluştur (110 adet)
- [ ] Function Code 0x03 (Read Holding Registers) desteği ekle
- [ ] **Test:** QModMaster ile bağlan → register değerlerini oku

### 3.3 LoRa → Register Eşleme (2 gün)

- [ ] LoRa'dan gelen sensör verisini ilgili register'a yaz
- [ ] Node ID'ye göre register offset hesapla: `base = (node_id - 1) * 10`
- [ ] Sistem register'larını güncelle (reg 100–109: aktif node sayısı, uptime, vb.)
- [ ] Son görülme zamanını takip et (node offline algılama)
- [ ] **Test:** Node sensör verisi gönder → QModMaster'da değer güncelleniyor mu?

### 3.4 Multi-Client Desteği (1 gün)

- [ ] Aynı anda 4 TCP bağlantıyı destekle
- [ ] Aktif bağlantı sayısını register 105'e yaz
- [ ] **Test:** 2 farklı Modbus istemcisi aynı anda bağlanıp okuyabiliyor mu?

**✅ Faz 3 Çıktısı:** Node → LoRa → Gateway → Modbus TCP → SCADA akışı çalışıyor, çoklu istemci desteği var.

---

## Faz 4 — Konsensüs Mekanizması (1 Hafta)

### 4.1 Zaman Dilimi Tabanlı İletim (3 gün)

- [ ] `consensus.h` / `consensus.cpp` oluştur
- [ ] Her node'a `config.h`'den sıra numarası ata
- [ ] Toplam node sayısını ve dilim süresini (10 sn) tanımla
- [ ] Node: `millis() / dilim_süresi % toplam_node == benim_sıram` → gönder
- [ ] Node: kendi dilimi değilse sessiz kal (RF yayını yapma)
- [ ] **Test:** 2 node, 10'ar saniyelik dilimlerle sırayla gönderiyor mu?

### 4.2 Hata Toleransı (2 gün)

- [ ] Gateway: bir node'dan veri gelmezse "offline" olarak işaretle
- [ ] Offline node'un dilimi atlanır, sistem kesintisiz devam eder
- [ ] Node tekrar çevrimiçi olduğunda sıraya geri döner
- [ ] **Test:** Bir node'u kapat → diğer node etkilenmeden devam ediyor mu?

**✅ Faz 4 Çıktısı:** Çakışmasız, adil bant genişliği paylaşımlı LoRa iletişimi.

---

## Faz 5 — Web Hata Ayıklama Paneli (1.5 Hafta)

### 5.1 ESP32 Web Sunucu (2 gün)

- [ ] `ESPAsyncWebServer` kütüphanesini ekle
- [ ] `web_dashboard.h` / `web_dashboard.cpp` oluştur
- [ ] Port 80'de HTTP sunucu başlat
- [ ] Ana sayfa: basit HTML ile sistem durumunu göster
- [ ] **Test:** Tarayıcıdan `http://<gateway_ip>` açılıyor mu?

### 5.2 Canlı Veri Sayfası (3 gün)

- [ ] Register değerleri tablosu (JavaScript ile 5 saniyede bir otomatik yenile)
- [ ] Node sağlık durumu (Online/Offline renkli gösterim)
- [ ] Son 50 LoRa paket logu (zaman, node ID, RSSI, veri)
- [ ] Sistem istatistikleri (uptime, bellek, toplam/hatalı paket)
- [ ] **Test:** Node veri gönderdiğinde web panelde anlık görünüyor mu?

### 5.3 REST API (2 gün)

- [ ] `GET /api/registers` → tüm register değerlerini JSON döndür
- [ ] `GET /api/nodes` → node listesi ve durumları
- [ ] `GET /api/stats` → sistem istatistikleri
- [ ] **Test:** Tarayıcıdan veya curl ile API çağrısı çalışıyor mu?

**✅ Faz 5 Çıktısı:** Tarayıcıdan erişilebilir canlı izleme paneli ve REST API.

---

## Faz 6 — Simülasyon ve Test Araçları (1.5 Hafta)

### 6.1 C# ASP.NET Web Projesi Kurulumu (1 gün)

- [ ] `simulation/` klasöründe `dotnet new web` ile proje oluştur
- [ ] NuGet paketleri: `NModbus4` veya `FluentModbus` (Modbus TCP istemci)
- [ ] `dotnet run` ile localhost:5000'de çalıştır

### 6.2 Modbus TCP Test İstemcisi (3 gün)

- [ ] Web arayüzünden Gateway IP ve port gir
- [ ] Register okuma sayfası (adres aralığı seç → değerleri tablo olarak göster)
- [ ] Otomatik polling (her N saniyede oku, grafik çiz)
- [ ] Bağlantı durumu göstergesi
- [ ] **Test:** Gateway'e bağlanıp register değerlerini web'den okuyor mu?

### 6.3 Ağ Topolojisi Görselleştirme (2 gün)

- [ ] Node'ları ve gateway'i web sayfasında diyagram olarak göster
- [ ] Her node'un durumunu (online/offline) ve RSSI değerini göster
- [ ] Konsensüs zaman dilimi gösterimi (şu an hangi node gönderiyor?)

### 6.4 Konsensüs Simülasyonu (2 gün)

- [ ] Gateway olmadan konsensüs algoritmasını simüle et
- [ ] N node, M dilim boyunca çalıştır
- [ ] Çakışma sayısı, adil paylaşım oranı çıktıları
- [ ] Sonuçları grafik olarak web sayfasında göster

**✅ Faz 6 Çıktısı:** Lokal web tabanlı Modbus test aracı, ağ topolojisi ve konsensüs simülasyonu.

---

## Faz 7 — Entegrasyon Testi ve Saha Deneyi (1 Hafta)

### 7.1 Masa Üstü Tam Sistem Testi (2 gün)

- [ ] Tüm bileşenleri bir arada çalıştır (2 node + 1 gateway)
- [ ] Node → LoRa (şifreli) → Gateway → Modbus TCP → QModMaster
- [ ] Aynı anda web panelden ve Modbus'tan izle
- [ ] 1 saat kesintisiz çalışma testi (stabilite)
- [ ] Hata senaryoları: node kapat/aç, Wi-Fi kes/bağla, yanlış anahtar gönder

### 7.2 Menzil Testi (2 gün)

- [ ] Açık alanda mesafe testi: 100m, 500m, 1km, 2km
- [ ] Her mesafede RSSI, SNR ve paket kayıp oranını kaydet
- [ ] SF7 vs SF10 vs SF12 menzil karşılaştırması
- [ ] Sonuçları tablo/grafik olarak belgele

### 7.3 Stres Testi (1 gün)

- [ ] Tüm node'ları aynı anda gönderime zorla (konsensüs olmadan) → çakışma oranı ölç
- [ ] Konsensüs ile tekrarla → çakışma sıfır mı?
- [ ] Modbus tarafında 4 eşzamanlı istemci ile sürekli okuma → gateway kilitlenmiyor mu?

**✅ Faz 7 Çıktısı:** Test sonuçları ile doğrulanmış, çalışan prototip.

---

## Faz 8 — Dokümantasyon ve Yayın (1 Hafta)

### 8.1 Teknik Dokümantasyon (3 gün)

- [ ] `docs/register_map.md` — tüm register adresleri, veri tipleri, açıklamalar
- [ ] `docs/hardware_setup.md` — fotoğraflı bağlantı rehberi
- [ ] `docs/protocol_flow.md` — uçtan uca veri akış diyagramı
- [ ] `docs/security_spec.md` — güvenlik katmanı teknik detayları

### 8.2 README.md (1 gün)

- [ ] İngilizce profesyonel README yaz
- [ ] Mimari diyagram, özellik listesi, kurulum adımları
- [ ] Badge'ler ekle (PlatformIO, License, Build Status)
- [ ] Demo GIF veya video linki

### 8.3 Demo ve Paylaşım (2 gün)

- [ ] 30 saniyelik demo videosu çek (node → gateway → SCADA akışı)
- [ ] LinkedIn paylaşım metni hazırla
- [ ] GitHub repo'yu public yap
- [ ] CV'ye ekle

**✅ Faz 8 Çıktısı:** Tamamen belgelenmiş, GitHub'da yayınlanmış, LinkedIn'de paylaşılmış proje.

---

## 📊 Toplam Zaman Özeti

| Faz | Süre | Kümülatif |
|-----|------|-----------|
| Faz 0 — Hazırlık | 1 hafta | 1. hafta |
| Faz 1 — LoRa Haberleşme | 2 hafta | 3. hafta |
| Faz 2 — Güvenlik Katmanı | 1.5 hafta | 4.5. hafta |
| Faz 3 — Modbus TCP | 1.5 hafta | 6. hafta |
| Faz 4 — Konsensüs | 1 hafta | 7. hafta |
| Faz 5 — Web Panel | 1.5 hafta | 8.5. hafta |
| Faz 6 — Simülasyon | 1.5 hafta | 10. hafta |
| Faz 7 — Test | 1 hafta | 11. hafta |
| Faz 8 — Dokümantasyon | 1 hafta | **12. hafta** |

> **Toplam: ~12 hafta (3 ay)** — haftada 10–15 saat çalışma varsayımıyla.

---

## ⚠️ Risk ve Önlemler

| Risk | Olasılık | Etki | Önlem |
|------|----------|------|-------|
| SX1278 modül arızası | Orta | Yüksek | Yedek modül al (+1 adet) |
| LoRa menzil yetersizliği | Düşük | Orta | SF artır, anten yükselt, LNA ekle |
| ESP32 bellek taşması | Orta | Yüksek | String yerine buffer kullan, heap monitör et |
| Wi-Fi kararsızlığı | Orta | Orta | Watchdog + otomatik reconnect |
| Modbus istemci uyumsuzluğu | Düşük | Düşük | Farklı SCADA yazılımlarıyla test et |
| Konsensüs senkron bozulması | Orta | Orta | NTP zaman senkronizasyonu ekle |

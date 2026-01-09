# SENSE — Smart Wearable for Safety Monitoring

> **Project**: SENSE (Smart Wearable for Safety Monitoring)
>
> **Target**: PKM-KC / PIMNAS
>
> **Form factor**: Jam tangan wearable, tanpa layar (audio + haptic + LED)
>
> **Arsitektur utama**: ESP32-S3-MINI-1-N8 + TinyML + Sensor cluster (MAX30102, BME688, LSM6DS3)

---

## Struktur README

1. Tujuan & Use-cases
2. Ringkasan Arsitektur
3. Bill of Materials (BoM) Utama
4. Analisis Komponen (keputusan pemilihan)
5. Power Management & Perkiraan Daya
6. Desain Mekanik & Ventilasi
7. Integrasi Perangkat Lunak (TinyML, Audio, Storage)
8. Catatan Prototyping & Pengujian
9. Risiko & Mitigasi
10. Roadmap dan Rekomendasi

---

## 1. Tujuan & Use-cases

* Deteksi kondisi medis kritis: HR tinggi/ rendah, SpO2 rendah (deteksi kelelahan/hipoksia).
* Deteksi lingkungan berbahaya: polutan/gas abnormal, temperatur ekstrem.
* Deteksi kejadian keselamatan: jatuh (man-down) melalui IMU.
* Pemberitahuan tak bergantung pada layar: audio (voice alerts), haptic (vibration), LED (status).
* Komunikasi ke gateway lokal via BLE/Wi‑Fi untuk logging dan alarm pusat.

---

## 2. Ringkasan Arsitektur

* **MCU**: ESP32-S3-MINI-1-N8 (Dual-core LX7, 240MHz, 8MB flash)
* **Inferensi Edge**: TFLite Micro (TinyML) untuk classifier: deteksi jatuh, deteksi anomali HR/SpO2, klasifikasi pola gas (preprocessing dari BME688)
* **Audio**: I2S DAC (MAX98357A) + speaker micro (CUI 15×11)
* **Haptic**: Coin ERM motor 10mm + MOSFET driver
* **Sensor**: MAX30102 (HR/SpO2), BME688 (IAQ/gas/temp/humidity/pressure), LSM6DS3 (6-axis IMU)
* **Power**: Li‑Po 350mAh, MCP73831 charger, LDO 3.3V (≥500mA) & 1.8V low-noise

---

## 3. Bill of Materials (BoM) Utama

| Kategori   |                                Komponen | Catatan                                        |
| ---------- | --------------------------------------: | ---------------------------------------------- |
| MCU        |                      ESP32-S3-MINI-1-N8 | 8 MB flash untuk model + audio file. WiFi/BLE. |
| Audio      |                     MAX98357A (I2S DAC) | Class-D, Hi‑Fi I2S to speaker                  |
| Speaker    |      CUI CMS-151135-076SP (15×11×3.5mm) | Micro dynamic speaker                          |
| Haptic     | Coin motor 1027 + MOSFET AO3400 + diode | ERM coin, driver diperlukan                    |
| Bio Sensor |                                MAX30102 | HR & SpO2, butuh 1.8V logic rail               |
| Env Sensor |                                  BME688 | Gas + IAQ + T/H/Pressure                       |
| IMU        |                                 LSM6DS3 | Accelerometer + Gyro                           |
| Charger    |                                MCP73831 | Li‑Po charge controller (CC/CV)                |
| Regulators |   LDO 3.3V (≥500mA), LDO 1.8V low-noise | 1.8V khusus untuk MAX30102                     |
| Battery    |                            Li‑Po 350mAh | Target profil pemakaian wearable               |

---

## 4. Analisis Komponen

> **MCU — ESP32‑S3‑MINI‑1‑N8**

* Memiliki vector instructions & cukup RAM/flash untuk menampung model TinyML dan beberapa file audio (.wav) di SPIFFS/Flash.
* WiFi 2.4GHz & BLE 5.0 cocok untuk mengirim data ke gateway lokal. Pilihan terbaik dibanding ESP8266 atau ESP32 standar.

> **Audio Subsystem**

* **MAX98357A** (I2S DAC) memungkinkan suara jernih dari file audio meski tanpa SD card. Menggunakan SPIFFS untuk menyimpan beberapa file peringatan.
* **CUI speaker** ukuran micro menyediakan output cukup terdengar saat dipadukan dengan Class-D amp.

> **Haptic + LED**

* Coin ERM 10mm dipilih karena profil tipis (muat di casing jam tangan). Driver MOSFET penting untuk menjaga kestabilan GPIO dan proteksi.
* LED SMD 0603 bi‑color untuk indikator minimal (status normal, warning, danger).

> **Sensor Cluster**

* **MAX30102**: akurasi HR/SpO2 memerlukan jalur 1.8V dan desain PCB yang baik agar noise rendah.
* **BME688**: kemampuan deteksi gas + AI classifier on-sensor (Bosch) berguna untuk membedakan sumber bau/gas di tambang.
* **LSM6DS3**: hemat daya dan fungsi pedometer internal mengurangi beban inferensi di MCU.

> **Komponen yang Dihapus**

* LCD, GSM module (SIM7000A), DFPlayer, DS1307 dihapus karena alasan daya, ukuran, dan reliability di lingkungan tambang.

---

## 5. Power Management & Perkiraan Daya (Estimasi sederhana)

> **Asumsi mode typical use**: wake-on-sensor (intermittent sampling), TinyML inferensi periodik, BLE advertising every few seconds, occasional WiFi upload.

* Battery: Li‑Po 350mAh (3.7V nominal)
* MCU (esp32-s3) average standby (light sleep + periodic wake): ~2–15 mA tergantung konfigurasi; peak TX WiFi: 200–400 mA (short bursts).
* Sensors idle + I2C: few 100s μA hingga 1–2 mA
* Speaker (peringatan): pulsed, bergantung volume — amp dapat draw besar saat aktif (beberapa 100 mA selama alarm)
* Haptic: 70–120 mA saat aktif (singkat)

**Rekomendasi**:

* Optimasi firmware untuk meminimalkan WiFi transmit (batch uploads) dan gunakan BLE beaconing saat tersedia.
* Gunakan deep-sleep modes dan memanfaatkan pedometer/interrupt di IMU untuk wake-on-movement.
* Atur charger MCP73831 untuk arus pengisian ~200–300 mA untuk keamanan baterai.

---

## 6. Desain Mekanik & Ventilasi

* **Ventilasi casing**: BME688 sensitif terhadap kelembaban lokal / keringat. Rancang bukaan ventilasi udara di sisi casing agar sensor tidak kontak langsung dengan kulit.
* **Posisi sensor HR**: MAX30102 harus menempel pada sisi casing yang relatif rata di area pergelangan, namun berikan buffer agar tidak selalu tertutup keringat (desain gasket / lubang minimal).
* **Isolasi elektronik**: tutup speaker dan motor dengan peredam agar getaran tidak menyebabkan noise ke sensor optik.
* **Material**: gunakan ABS/PC untuk casing atau material yang tahan benturan dan tidak mudah retak di kondisi tambang.

---

## 7. Integrasi Perangkat Lunak

* **TinyML (TFLite Micro)**: model untuk deteksi jatuh dan anomali HR/SpO2. Model pruned & quantized (int8) agar muat di 8MB flash dan infer real-time.
* **Audio**: file .wav pendek (mono, 8–16 kHz) disimpan di SPIFFS; MAX98357A dibunyikan melalui I2S playback.
* **Data storage**: ring buffer di flash untuk log kritikal; upload ke gateway saat koneksi stabil.
* **Time sync**: gunakan NTP saat WiFi tersedia, tidak gunakan DS1307 eksternal.
* **OTA**: siapkan mekanisme OTA terbatas (untuk field updates) via WiFi.

---

## 8. Catatan Prototyping & Pengujian

* Lakukan pengujian noise optik untuk MAX30102 pada beberapa tipe kulit dan kondisi berkeringat.
* Uji sensitivitas BME688 terhadap sumber gas yang relevan di tambang (dengan simulasi aman di lab).
* Validasi deteksi jatuh (LSM6DS3) dengan dataset kecil: jatuh simulasi vs aktivitas normal (jatuh palsu minim).
* Uji endurance baterai di mode nyata (shift 12 jam) dengan skenario alarm.

---

## 9. Risiko & Mitigasi

* **Sinyal seluler buruk**: gunakan gateway lokal dan BLE/WiFi mesh untuk menghindari ketergantungan GSM.
* **Kontaminasi sensor**: ventilasi casing dan prosedur maintenance berkala.
* **Kebisingan sinyal optik**: desain optik dan PCB yang baik (jalur 1.8V low-noise, ground plane).
* **Daya puncak WiFi**: batasi upload, gunakan batching dan QoS untuk alarm kritis.

---


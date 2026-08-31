# Pi Ses Sistemi

SVAN 971 ses ölçer ile alınan akustik ölçümleri Raspberry Pi üzerinden yöneten, kayıt tamamlandığında ham SVL dosyasını cihazdan indirip WAV ve CSV çıktıları üreten, bu üç dosyayı Raspberry Pi üzerinde AES-256-GCM ile şifreleyerek merkezi bir FastAPI backend'ine gönderen web tabanlı bir ölçüm sistemidir.

Sistem üç farklı çalışma biçimini destekler: kullanıcı tarafından başlatılıp durdurulan normal kayıt, yalnızca anlık dB takibi yapan canlı izleme ve belirlenen ses eşiğine göre otomatik kayıt oluşturan eşik modu. Eşik modu, kayıt backend tarafından alındıktan sonra isteğe bağlı bir cooldown süresi bekleyip aynı ölçüm setup'ını yeniden yükleyerek sürekli çalışabilir.

Bu dosyanın ilk yarısı Türkçe, ikinci yarısı İngilizcedir.

---

# Türkçe

## Sistem ne yapar?

SVAN 971, Raspberry Pi'ye USB üzerinden bağlanır. Raspberry Pi üzerinde çalışan edge agent cihazın ölçüm ve logger durumunu yönetir, SVAN SD kartında oluşan son `.SVL` dosyasını indirir ve proje içindeki dönüştürme araçlarıyla iki ek çıktı üretir:

```text
raw.SVL       SVAN cihazından indirilen ham kayıt
audio.wav     Web panelinde dinlemek ve waveform üretmek için ses çıktısı
data_all.csv  Ölçüm, logger, spektrum ve waveform verilerini birleştiren CSV
```

Bu dosyalar Raspberry Pi üzerinde AES-256-GCM ile ayrı ayrı şifrelenir:

```text
raw.SVL.enc
audio.wav.enc
data_all.csv.enc
```

Ana backend yalnızca şifreli edge upload kabul eder. Gelen dosyaları aynı AES anahtarıyla doğrular, kalıcı depolamaya `.enc` biçiminde kaydeder ve veritabanındaki kaydı doğrudan `encrypted` durumunda oluşturur. Web paneli ses oynatma, waveform ve CSV grafikleri gerektiğinde ilgili dosyayı geçici olarak çözer; kalıcı plaintext kopya oluşturmaz.

## Mimari

```text
SVAN 971
   │ USB
   ▼
Raspberry Pi
   edge_agent/
   backend/tools/
   geçici çalışma klasörü
   │
   │ AES-256-GCM ile şifreli HTTP upload
   ▼
Windows PC veya merkezi sunucu
   backend/      FastAPI
   frontend/     React + Vite
   recordings/   şifreli kayıt deposu
   recordings.db SQLite veritabanı
```

Raspberry Pi kalıcı kayıt arşivi olarak kullanılmaz. Dönüştürme sırasında plaintext WAV, CSV ve SVL dosyaları edge çalışma klasöründe geçici olarak bulunur. Upload başarıyla tamamlandığında ve `DELETE_AFTER_UPLOAD=1` olduğunda tüm geçici oturum klasörü silinir. Upload başarısız olursa tekrar denemeye izin vermek için oturum ve geçici dosyalar korunabilir.

## Çalışma modları

### Normal kayıt

Web panelindeki kayıt düğmesi normal kayıt akışını başlatır. Cihaz daha önce bir AUTO setup'ında kalmış olsa bile edge agent önce ölçümü durdurur, `RECORD` setup'ını yükler, logger'ı açar ve ölçümü başlatır:

```text
S0
#7,LS,RECORD
T1
S1
```

Kullanıcı kaydı durdurduğunda edge agent son logger adını okur, ilgili SVL dosyasını indirir, WAV ve CSV üretir, bütün dosyaları şifreler ve backend'e gönderir.

`MANUAL_RECORD_SETUP` ortam değişkeni varsayılan olarak `RECORD` değerini kullanır. Setup adı en fazla 8 karakter olmalıdır.

### Canlı izleme

Canlı izleme modu anlık ses seviyesini web panelinde gösterir ancak kayıt oluşturmaz. Bu modda logger kapatılır:

```text
T0
S1
```

Canlı izleme durdurulduğunda cihaz tekrar stop durumuna alınır ve logger yeniden açılır.

### Eşik tabanlı otomatik kayıt

Eşik modu panelde girilen kısa süreli SPL değerini izler. Ölçüm değeri eşik üzerinde belirlenen `trigger hold` süresi boyunca kalırsa olay tetiklenir. Tetiklemeden sonra değer eşik altında belirlenen `release hold` süresi boyunca kalırsa kayıt otomatik olarak sonlandırılır, indirilir, dönüştürülür, şifrelenir ve backend'e gönderilir.

Eşik modu başlamadan önce edge agent panelde seçilen SVAN setup'ını yükler:

```text
S0
#7,LS,<AUTO_SETUP>
T1
S1
```

Panel ve cihaz setup'ındaki eşik değerleri birbiriyle uyumlu olmalıdır. Yazılım tarafındaki tetik kararı kısa süreli anlık SPL üzerinden verilir; waveform'un gerçekten SVL içine yazılması ise SVAN üzerindeki Event Recording ayarına bağlıdır.

Otomatik yeniden kurma seçeneği açıksa akış şöyledir:

```text
armed
→ threshold triggered
→ release hold
→ stop/download/extract/encrypt/upload
→ backend acknowledgement
→ cooldown
→ same setup + T1 + S1
→ armed
```

Her döngü ayrı bir logger dosyası ve ayrı bir backend kaydı oluşturur. Finalizasyon, upload, cooldown ve yeniden başlatma sırasında kısa kayıt boşlukları oluşabilir.

## SVAN 971 setup hazırlığı

Sistem başlamadan önce cihaz üzerinde en az iki setup hazırlanmalıdır.

### RECORD setup

Normal kayıt için önerilen temel yapı:

```text
Setup adı: RECORD
Logger: On
Logger Trigger: Off
Measurement Trigger: Off
Event Recording: Continuous
Logger Step: 1 s
```

Bu setup'ta waveform ölçüm başlar başlamaz yazılmalı ve `S0` komutuna kadar devam etmelidir. Event Recording filtresi analiz ihtiyacına göre A veya Z seçilebilir. CSV tarafında kullanılacak profil ve logger sonuçları cihaz üzerindeki ölçüm amacına göre ayarlanmalıdır.

### AUTO setup

Eşik modu için cihaz üzerinde panelde girilecek adla eşleşen bir setup oluşturulmalıdır. Örnek:

```text
AUTO60
AUTO80
AUTO120
```

Setup adı en fazla 8 karakter olabilir. AUTO setup'ta Event Recording cihaz tarafında `Level+` benzeri eşik kontrollü moda ayarlanmalı, threshold değeri panelde kullanılan değerle eşleştirilmeli ve gerekiyorsa pre-trigger etkinleştirilmelidir.

Edge agent setup oluşturmaz; yalnızca cihaz üzerinde önceden kaydedilmiş setup'ı yükler.

## Gereksinimler

### Donanım

- SVAN 971 ses ölçer
- Raspberry Pi veya geliştirme için Ubuntu sanal makinesi
- SVAN ile uyumlu USB bağlantısı
- Windows PC ya da FastAPI backend ve React frontend çalıştırabilecek bir sunucu
- Aynı yerel ağa bağlı edge ve backend cihazları

### Windows yazılımı

- Git
- Python 3.12 önerilir
- Node.js LTS ve npm
- PowerShell

### Raspberry Pi / Ubuntu yazılımı

- Python 3
- `python3-venv`
- `libusb-1.0-0`
- `usbutils`
- Git

## Kurulum

Aşağıdaki adımlar projenin `main` branch'inin son sürümü içindir.

### 1. Depoyu Windows bilgisayara klonlayın

```powershell
git clone https://github.com/emirayar/pi-ses-sistemi.git
cd pi-ses-sistemi
git switch main
```

### 2. Windows backend ortamını hazırlayın

Proje kökünde:

```powershell
py -3.12 -m venv backend\venv
backend\venv\Scripts\python.exe -m pip install --upgrade pip
backend\venv\Scripts\python.exe -m pip install -r backend\requirements.txt
```

PowerShell script çalıştırma politikası engel olursa yalnızca açık terminal için izin verin:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

### 3. Frontend bağımlılıklarını kurun

```powershell
cd frontend
npm install
cd ..
```

### 4. Depoyu Raspberry Pi'ye klonlayın

```bash
cd ~
git clone https://github.com/emirayar/pi-ses-sistemi.git
cd pi-ses-sistemi
git switch main
```

### 5. Raspberry Pi sistem paketlerini kurun

```bash
sudo apt update
sudo apt install -y git python3 python3-venv libusb-1.0-0 usbutils
```

Edge Python ortamını oluşturun:

```bash
cd ~/pi-ses-sistemi
python3 -m venv backend/venv
backend/venv/bin/python -m pip install --upgrade pip
backend/venv/bin/python -m pip install -r backend/edge_requirements.txt
```

`cryptography` kurulumu kaynak derleme hatası verirse şu paketler gerekebilir:

```bash
sudo apt install -y build-essential libssl-dev libffi-dev python3-dev
backend/venv/bin/python -m pip install -r backend/edge_requirements.txt
```

### 6. SVAN USB bağlantısını doğrulayın

```bash
lsusb
```

SVAN cihazının USB kimliği görünmelidir. Projede kullanılan cihaz için filtreli kontrol:

```bash
lsusb | grep 0017
```

Bağlantı testi:

```bash
cd ~/pi-ses-sistemi
sudo backend/venv/bin/python backend/tools/svantek_control.py --status
```

Cihaz görünmüyorsa kabloyu, SVAN USB ayarını ve sanal makine kullanılıyorsa USB passthrough ayarını kontrol edin. VirtualBox içinde cihazın VM'e bağlanması gerekir.

### 7. AES anahtarı üretin

Anahtarı bir kez üretin:

```powershell
backend\venv\Scripts\python.exe backend\tools\generate_aes_key.py
```

Çıktı, 32 byte AES anahtarının base64 karşılığıdır. Aynı değer hem Windows backend'de hem Raspberry Pi edge agent'ta kullanılmalıdır.

Bu anahtarı kaybederseniz mevcut `.enc` kayıtlar çözülemez. Anahtarı Git'e, README'ye, ekran görüntüsüne veya paylaşılan scriptlere yazmayın.

### 8. Yerel başlangıç scriptlerini oluşturun

Windows'ta:

```powershell
Copy-Item scripts\start_backend_keyed.example.ps1 start_backend_keyed.local.ps1
Copy-Item scripts\start_frontend.example.ps1 start_frontend.local.ps1
```

Dosyaları düzenleyin:

```powershell
notepad start_backend_keyed.local.ps1
notepad start_frontend.local.ps1
```

`start_backend_keyed.local.ps1` içinde şu iki değer doğru olmalıdır:

```powershell
$env:AES_KEY_B64 = "PI_ILE_AYNI_AES_KEY"
$env:EDGE_BASE_URL = "http://PI_IP:8010"
```

Raspberry Pi'de:

```bash
cd ~/pi-ses-sistemi
cp scripts/start_edge_agent.example.sh start_edge_agent.local.sh
chmod +x start_edge_agent.local.sh
nano start_edge_agent.local.sh
```

Yerel edge scriptinde en az şu değerleri düzenleyin:

```bash
MAIN_BACKEND_URL="http://WINDOWS_IP:8000"
AES_KEY_B64="WINDOWS_ILE_AYNI_AES_KEY"
MANUAL_RECORD_SETUP="RECORD"
EDGE_ENCRYPTION_REQUIRED="1"
SVAN_SAMPLE_RATE="8000"
```

`SVAN_SAMPLE_RATE`, cihazdaki Event Recording örnekleme hızıyla ve SVL dönüştürme akışıyla uyumlu olmalıdır. Setup 12 kHz kullanıyorsa bu değeri `12000` olarak değiştirin.

`.local.ps1` ve `.local.sh` dosyaları Git tarafından yok sayılır. Gerçek IP adreslerini ve AES anahtarını yalnızca bu yerel dosyalarda tutun.

### 9. Ağ ayarlarını yapın

Windows IP adresini öğrenin:

```powershell
ipconfig
```

Raspberry Pi IP adresini öğrenin:

```bash
hostname -I
```

Windows backend'in 8000 portuna yerel ağdan erişilebilmesi gerekir. Gerekirse yönetici PowerShell'de firewall kuralı ekleyin:

```powershell
New-NetFirewallRule `
  -DisplayName "Pi Ses Backend 8000" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 8000 `
  -Action Allow
```

Frontend başka bir cihazdan açılacaksa Vite'ın yayınladığı ağ adresini kullanabilirsiniz. Varsayılan geliştirme adresi:

```text
http://localhost:5173
```

### 10. Sistemi başlatın

Önerilen sıra:

Windows backend:

```powershell
.\start_backend_keyed.local.ps1
```

Raspberry Pi edge agent:

```bash
cd ~/pi-ses-sistemi
./start_edge_agent.local.sh
```

Windows frontend:

```powershell
.\start_frontend.local.ps1
```

Tarayıcı:

```text
http://localhost:5173
```

## Kurulum kontrolü

### Edge agent durumu

Windows PowerShell'den:

```powershell
curl.exe -m 15 http://PI_IP:8010/api/edge/status
```

Başarılı durumda özellikle şu alanları kontrol edin:

```json
{
  "status": "ok",
  "svan_ok": true,
  "pi_side_encryption_required": true,
  "encryption_key_configured": true,
  "encryption_algorithm": "AES-256-GCM",
  "manual_record_setup": "RECORD"
}
```

### Backend bağlantısı

```powershell
curl.exe -m 15 http://127.0.0.1:8000/api/recordings
curl.exe -m 15 http://127.0.0.1:8000/api/recording-session/status
```

Raspberry Pi'den backend erişimi:

```bash
curl -m 15 http://WINDOWS_IP:8000/api/recordings
```

### Kod kontrolü

Windows:

```powershell
cd C:\path\to\pi-ses-sistemi

git diff --check

backend\venv\Scripts\python.exe -m compileall backend edge_agent

cd frontend
npm run build
cd ..
```

Raspberry Pi:

```bash
cd ~/pi-ses-sistemi
backend/venv/bin/python -m py_compile edge_agent/main.py
```

## Kullanım

### Normal kayıt alma

1. Web panelini açın.
2. Kayıt bilgilerini girin.
3. Kayıt düğmesine basın.
4. Edge agent `RECORD` setup'ını yükler ve ölçümü başlatır.
5. Kayıt tamamlandığında durdurma düğmesine basın.
6. SVL indirme, WAV/CSV çıkarma, şifreleme ve upload işlemi tamamlanana kadar bekleyin.

Yeni kayıt listede `encrypted` durumunda görünmelidir.

### Canlı dB izleme

Canlı ölçüm panelinde threshold seçeneğini kapalı tutarak izlemeyi başlatın. Bu mod dosya veya backend kaydı üretmez.

### Otomatik eşik kaydı

1. SVAN üzerinde kullanılacak AUTO setup'ını önceden hazırlayın.
2. Panelde threshold seçeneğini açın.
3. Cihaz setup adını, dB eşiğini, trigger hold ve release hold sürelerini girin.
4. Sürekli çalışma gerekiyorsa otomatik yeniden kurma seçeneğini açıp cooldown süresini belirleyin.
5. İzlemeyi başlatın.

Kayıt tamamlandığında backend cevabından sonra cooldown başlar. Cooldown bitince edge agent aynı setup'ı yeniden yükler ve yeni bir logger oturumu oluşturur.

## Kayıtların saklanması

Her kayıt backend içinde ayrı bir klasörde tutulur:

```text
backend/recordings/
  <timestamp_title>/
    audio.wav.enc
    data_all.csv.enc
    raw.SVL.enc
```

Veritabanında plaintext yol alanları dosyanın mantıksal adını koruyabilir; Pi üzerinden gelen kayıtlar için kalıcı plaintext dosya bulunmaz. Panel ses veya CSV istediğinde şifreli dosya geçici olarak açılır.

`data_all.csv`, farklı ölçüm görünümlerini tek dosyada tutan long-format bir yapıdır. `view`, `metric`, `band_hz`, `time_sec`, `value` ve `unit` gibi alanlar üzerinden waveform ve ölçüm grafikleri oluşturulur.

## Proje yapısı

```text
backend/
  main.py
  database.py
  models.py
  requirements.txt
  edge_requirements.txt
  routers/
  services/
    encryption_service.py
  tools/
    svantek_control.py
    svantek_live_once.py
    svantek_ls.py
    svantek_download_file.py
    svl_extract_wav_raw24.py
    svl_extract_all_csv.py
    generate_aes_key.py

edge_agent/
  main.py

frontend/
  src/
  package.json
  vite.config.js

scripts/
  start_backend_keyed.example.ps1
  start_edge_agent.example.sh
  start_frontend.example.ps1
```

## Güvenlik ve Git notları

Aşağıdaki dosyalar repoya gönderilmemelidir:

```text
.env
*.local.ps1
*.local.sh
backend/recordings/
backend/downloads/
backend/recordings.db
backend/venv/
frontend/node_modules/
frontend/dist/
*.wav
*.csv
*.SVL
*.enc
```

AES anahtarı kaynak kodda tutulmamalıdır. `git add .` yerine değiştirilen dosyaları açıkça stage etmek daha güvenlidir.

Örnek:

```powershell
git add README.md
git add scripts/start_backend_keyed.example.ps1
git add scripts/start_edge_agent.example.sh
git add scripts/start_frontend.example.ps1
```

## Sorun giderme

### `encryption_key_configured` false

Pi başlangıç scriptindeki `AES_KEY_B64` eksik veya edge agent'a aktarılmıyor. Windows ve Pi'deki anahtarların birebir aynı olduğundan emin olun.

### Backend şifreli upload'ı reddediyor

Yanlış AES anahtarı, bozuk upload veya backend ile Pi arasında farklı proje sürümü olabilir. Her iki cihazda da aynı `main` commit'ini kullanın.

### RECORD setup yüklenemiyor

SVAN üzerinde `RECORD` adında setup bulunduğunu, adın büyük-küçük harf ve karakter olarak eşleştiğini kontrol edin.

### Eşik tetikleniyor ancak WAV oluşmuyor

Yazılım threshold'u yalnızca workflow tetiklemek için kullanır. Waveform'un SVL içine yazılması cihazdaki AUTO setup'ın Event Recording ayarına bağlıdır. Setup eşiğini, Level+ ayarını ve pre-trigger yapılandırmasını kontrol edin.

### Ses süresi veya oynatma hızı yanlış

`SVAN_SAMPLE_RATE` değeri ile cihazın Event Recording sampling değeri uyuşmuyor olabilir. İki tarafı aynı değere getirin.

### Edge agent SVAN'a erişemiyor

`lsusb`, USB kablosu, cihaz USB hızı, VirtualBox USB passthrough ve edge agent'ın `sudo` ile çalıştığını kontrol edin.

### Stop işlemi gecikiyor

Canlı polling ve cihaz komutlarının aynı anda USB hattını kullanmadığından emin olun. Edge agent içindeki USB lock bu çakışmayı azaltır; eski bir branch veya eski çalışan process kalmadığını kontrol edin.

---

# English

## What does the system do?

Pi Ses Sistemi controls an SVAN 971 sound level meter through a Raspberry Pi connected over USB. The edge agent downloads the latest `.SVL` logger file from the instrument and produces two additional outputs:

```text
raw.SVL       Original logger file downloaded from the SVAN
audio.wav     Audio output used for playback and waveform rendering
data_all.csv  Combined measurement, logger, spectrum, and waveform data
```

All three files are encrypted separately on the Raspberry Pi with AES-256-GCM:

```text
raw.SVL.enc
audio.wav.enc
data_all.csv.enc
```

The main backend accepts encrypted edge uploads only. It validates every encrypted file with the same AES key, stores the `.enc` files permanently, and creates the database row directly with an `encrypted` status. When the web interface needs audio, waveform, or CSV data, the backend decrypts the required file temporarily without keeping a persistent plaintext copy.

## Architecture

```text
SVAN 971
   │ USB
   ▼
Raspberry Pi
   edge_agent/
   backend/tools/
   temporary work directory
   │
   │ AES-256-GCM encrypted HTTP upload
   ▼
Windows PC or central server
   backend/      FastAPI
   frontend/     React + Vite
   recordings/   encrypted recording storage
   recordings.db SQLite database
```

The Raspberry Pi is not intended to be permanent storage. Plain WAV, CSV, and SVL files exist temporarily in the edge work directory while conversion and encryption are performed. When the upload succeeds and `DELETE_AFTER_UPLOAD=1`, the whole temporary session directory is removed. If an upload fails, the session and temporary files may remain available for retry.

## Operating modes

### Manual recording

The Record button starts a normal recording. The edge agent does not trust the setup that was previously active on the device. It stops the instrument, loads the `RECORD` setup, enables the logger, and starts the measurement:

```text
S0
#7,LS,RECORD
T1
S1
```

When the user stops the recording, the edge agent reads the latest logger name, downloads the matching SVL file, creates WAV and CSV outputs, encrypts all files, and uploads them to the backend.

The default setup name is controlled by `MANUAL_RECORD_SETUP=RECORD`. SVAN setup names must be no longer than 8 characters.

### Live monitoring

Live monitoring displays the current sound level without creating a recording. The logger is disabled in this mode:

```text
T0
S1
```

When live monitoring stops, the instrument returns to the stopped state and the logger is enabled again.

### Threshold-triggered recording

Threshold mode observes the short-term SPL value shown by the live measurement endpoint. An event is triggered after the value remains above the configured threshold for the trigger-hold duration. After triggering, the recording is finalized when the value remains below the threshold for the release-hold duration.

Before threshold monitoring starts, the edge agent loads the setup selected in the panel:

```text
S0
#7,LS,<AUTO_SETUP>
T1
S1
```

The threshold value configured in the web panel should match the threshold stored in the SVAN setup. The software trigger controls the workflow, while actual waveform data inside the SVL file depends on the Event Recording configuration of the instrument.

With automatic re-arm enabled, the workflow is:

```text
armed
→ threshold triggered
→ release hold
→ stop/download/extract/encrypt/upload
→ backend acknowledgement
→ cooldown
→ same setup + T1 + S1
→ armed
```

Each cycle creates a separate logger file and a separate backend recording. Short gaps may occur during finalization, upload, cooldown, and device restart.

## Preparing SVAN 971 setups

At least two setups should be prepared on the instrument before using the system.

### RECORD setup

Recommended baseline:

```text
Setup name: RECORD
Logger: On
Logger Trigger: Off
Measurement Trigger: Off
Event Recording: Continuous
Logger Step: 1 s
```

Waveform recording should begin with the measurement and continue until the device receives `S0`. The Event Recording filter can be A or Z depending on the analysis requirements. Profiles and logger results should be configured for the intended measurement task.

### AUTO setup

Create one or more threshold setups with names that can be entered in the panel, for example:

```text
AUTO60
AUTO80
AUTO120
```

Setup names are limited to 8 characters. Event Recording should use an instrument-side threshold mode such as `Level+`, and its threshold should match the value used in the panel. Enable pre-trigger when required.

The edge agent does not create setups. It only loads setups that already exist on the instrument.

## Requirements

### Hardware

- SVAN 971 sound level meter
- Raspberry Pi, or an Ubuntu virtual machine for development
- A compatible USB connection to the SVAN
- A Windows PC or server capable of running FastAPI and React
- Network connectivity between the edge device and the backend

### Windows software

- Git
- Python 3.12 recommended
- Node.js LTS and npm
- PowerShell

### Raspberry Pi / Ubuntu software

- Python 3
- `python3-venv`
- `libusb-1.0-0`
- `usbutils`
- Git

## Installation

The following steps describe the final `main` branch.

### 1. Clone the repository on Windows

```powershell
git clone https://github.com/emirayar/pi-ses-sistemi.git
cd pi-ses-sistemi
git switch main
```

### 2. Create the Windows backend environment

From the project root:

```powershell
py -3.12 -m venv backend\venv
backend\venv\Scripts\python.exe -m pip install --upgrade pip
backend\venv\Scripts\python.exe -m pip install -r backend\requirements.txt
```

When PowerShell blocks local scripts, allow them only for the current terminal:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

### 3. Install frontend dependencies

```powershell
cd frontend
npm install
cd ..
```

### 4. Clone the repository on the Raspberry Pi

```bash
cd ~
git clone https://github.com/emirayar/pi-ses-sistemi.git
cd pi-ses-sistemi
git switch main
```

### 5. Install Raspberry Pi system packages

```bash
sudo apt update
sudo apt install -y git python3 python3-venv libusb-1.0-0 usbutils
```

Create the edge Python environment:

```bash
cd ~/pi-ses-sistemi
python3 -m venv backend/venv
backend/venv/bin/python -m pip install --upgrade pip
backend/venv/bin/python -m pip install -r backend/edge_requirements.txt
```

If `cryptography` must be built locally and installation fails:

```bash
sudo apt install -y build-essential libssl-dev libffi-dev python3-dev
backend/venv/bin/python -m pip install -r backend/edge_requirements.txt
```

### 6. Verify the SVAN USB connection

```bash
lsusb
lsusb | grep 0017
```

Test the instrument:

```bash
cd ~/pi-ses-sistemi
sudo backend/venv/bin/python backend/tools/svantek_control.py --status
```

If the device is not visible, check the cable, the USB mode on the SVAN, and USB passthrough when running inside VirtualBox.

### 7. Generate the AES key

Generate the key once on Windows:

```powershell
backend\venv\Scripts\python.exe backend\tools\generate_aes_key.py
```

Use the exact same value on the Windows backend and the Raspberry Pi edge agent.

Losing this key makes existing `.enc` recordings impossible to decrypt. Never commit it, paste it into the README, or store it in a shared screenshot.

### 8. Create local startup scripts

On Windows:

```powershell
Copy-Item scripts\start_backend_keyed.example.ps1 start_backend_keyed.local.ps1
Copy-Item scripts\start_frontend.example.ps1 start_frontend.local.ps1
```

Edit the local backend script:

```powershell
notepad start_backend_keyed.local.ps1
```

Required values:

```powershell
$env:AES_KEY_B64 = "THE_SAME_KEY_USED_ON_THE_PI"
$env:EDGE_BASE_URL = "http://PI_IP:8010"
```

On the Raspberry Pi:

```bash
cd ~/pi-ses-sistemi
cp scripts/start_edge_agent.example.sh start_edge_agent.local.sh
chmod +x start_edge_agent.local.sh
nano start_edge_agent.local.sh
```

Required values include:

```bash
MAIN_BACKEND_URL="http://WINDOWS_IP:8000"
AES_KEY_B64="THE_SAME_KEY_USED_ON_WINDOWS"
MANUAL_RECORD_SETUP="RECORD"
EDGE_ENCRYPTION_REQUIRED="1"
SVAN_SAMPLE_RATE="8000"
```

`SVAN_SAMPLE_RATE` must match the Event Recording sampling rate and the SVL extraction workflow. Use `12000` when the instrument setup records at 12 kHz.

Local scripts are ignored by Git. Keep real IP addresses and AES keys only in `.local.ps1` and `.local.sh` files.

### 9. Configure networking

Find the Windows address:

```powershell
ipconfig
```

Find the Raspberry Pi address:

```bash
hostname -I
```

Allow inbound TCP port 8000 on Windows when required:

```powershell
New-NetFirewallRule `
  -DisplayName "Pi Ses Backend 8000" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 8000 `
  -Action Allow
```

The default frontend development address is:

```text
http://localhost:5173
```

### 10. Start the system

Recommended order:

Windows backend:

```powershell
.\start_backend_keyed.local.ps1
```

Raspberry Pi edge agent:

```bash
cd ~/pi-ses-sistemi
./start_edge_agent.local.sh
```

Windows frontend:

```powershell
.\start_frontend.local.ps1
```

Open:

```text
http://localhost:5173
```

## Installation verification

### Edge status

```powershell
curl.exe -m 15 http://PI_IP:8010/api/edge/status
```

Important fields:

```json
{
  "status": "ok",
  "svan_ok": true,
  "pi_side_encryption_required": true,
  "encryption_key_configured": true,
  "encryption_algorithm": "AES-256-GCM",
  "manual_record_setup": "RECORD"
}
```

### Backend connectivity

```powershell
curl.exe -m 15 http://127.0.0.1:8000/api/recordings
curl.exe -m 15 http://127.0.0.1:8000/api/recording-session/status
```

From the Raspberry Pi:

```bash
curl -m 15 http://WINDOWS_IP:8000/api/recordings
```

### Source checks

Windows:

```powershell
git diff --check
backend\venv\Scripts\python.exe -m compileall backend edge_agent

cd frontend
npm run build
cd ..
```

Raspberry Pi:

```bash
backend/venv/bin/python -m py_compile edge_agent/main.py
```

## Usage

### Manual recording

Open the web panel, enter the recording metadata, and press Record. The edge agent loads the `RECORD` setup before starting the instrument. Press Stop when the measurement is complete and wait for SVL download, extraction, encryption, and upload to finish.

The new recording should appear with an `encrypted` status.

### Live dB monitoring

Start live monitoring with threshold mode disabled. This mode does not create files or a backend recording.

### Automatic threshold recording

Prepare the AUTO setup on the SVAN first. Enable threshold mode in the panel, enter the setup name, threshold, trigger-hold duration, and release-hold duration. Enable automatic re-arm and configure a cooldown when continuous event capture is required.

Cooldown starts after the backend acknowledges the encrypted upload. At the end of the cooldown, the edge agent loads the same setup and creates a new logger session.

## Recording storage

Each recording is stored in a separate backend directory:

```text
backend/recordings/
  <timestamp_title>/
    audio.wav.enc
    data_all.csv.enc
    raw.SVL.enc
```

Plain path columns may remain in the database as logical filenames, but no persistent plaintext files exist for recordings received from the Pi. The backend decrypts encrypted files temporarily when the panel requests audio or CSV data.

`data_all.csv` uses a long-format structure. Columns such as `view`, `metric`, `band_hz`, `time_sec`, `value`, and `unit` are used to render waveform and measurement charts.

## Project structure

```text
backend/
  main.py
  database.py
  models.py
  requirements.txt
  edge_requirements.txt
  routers/
  services/
    encryption_service.py
  tools/
    svantek_control.py
    svantek_live_once.py
    svantek_ls.py
    svantek_download_file.py
    svl_extract_wav_raw24.py
    svl_extract_all_csv.py
    generate_aes_key.py

edge_agent/
  main.py

frontend/
  src/
  package.json
  vite.config.js

scripts/
  start_backend_keyed.example.ps1
  start_edge_agent.example.sh
  start_frontend.example.ps1
```

## Security and Git notes

Do not commit runtime data, secrets, or local environments:

```text
.env
*.local.ps1
*.local.sh
backend/recordings/
backend/downloads/
backend/recordings.db
backend/venv/
frontend/node_modules/
frontend/dist/
*.wav
*.csv
*.SVL
*.enc
```

Keep the AES key outside the source code. Prefer explicitly staging changed files instead of using `git add .`.

## Troubleshooting

### `encryption_key_configured` is false

`AES_KEY_B64` is missing from the Pi startup environment or the edge process did not receive it. Verify that Windows and Pi use exactly the same key.

### The backend rejects the encrypted upload

The usual causes are different AES keys, corrupted upload data, or different project versions on Windows and the Pi. Pull the same `main` commit on both machines.

### The RECORD setup cannot be loaded

Confirm that a setup named `RECORD` exists on the SVAN and that `MANUAL_RECORD_SETUP` matches it exactly.

### The threshold triggers but no WAV is produced

The software threshold controls the workflow only. Actual waveform data inside the SVL file depends on the Event Recording settings of the AUTO setup. Check the instrument-side threshold, `Level+` mode, and pre-trigger configuration.

### Audio duration or playback speed is wrong

`SVAN_SAMPLE_RATE` may not match the Event Recording sampling rate. Configure both sides with the same value.

### The edge agent cannot access the SVAN

Check `lsusb`, the USB cable, the USB mode on the instrument, VirtualBox USB passthrough, and whether the edge agent is running through `sudo`.

### Stop is slow or unreliable

Make sure an old edge process is not still running. The current edge agent serializes SVAN USB operations to reduce collisions between live polling and stop commands.

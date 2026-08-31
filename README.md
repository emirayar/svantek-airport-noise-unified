# SVANTEK Airport Noise - Unified System

SVAN 971 ile alınan kayıtları güvenli şekilde merkezi panele aktaran ve
kayıt tamamlandıktan sonra Airport Noise Detection modelleriyle inceleyen
deneysel sistem.

## Ne yapar?

1. Raspberry Pi üzerindeki edge agent, SVAN 971 kaydını yönetir.
2. Kayıt bitince WAV, CSV ve ham SVL dosyalarını AES-256-GCM ile şifreleyip
   backend'e gönderir.
3. Backend şifreli kaydı kalıcı olarak `.enc` biçiminde tutar.
4. Analiz sırasında ses yalnız geçici olarak açılır, mono/22.050 Hz'e
   dönüştürülür ve sınıflandırılır.
5. Panel, algılanan olayları zaman çizelgesinde gösterir; olaya tıklanınca
   kayıt ilgili saniyeye gider.

Panel ayrıca daha önce alınmış `audio.wav.enc` kayıtlarını doğrudan yükleyebilir.
Varsa `data_all.csv.enc` ve `raw.SVL.enc` dosyaları aynı seçimde eklenebilir.

## Mimari

```text
SVAN 971 -> Raspberry Pi edge agent -> encrypted upload -> FastAPI backend
                                                     -> SQLite + encrypted storage
                                                     -> Airport AI analysis
React/Vite panel <---------------------------------- events, audio, CSV, waveform
```

## Kurulum

Python 3.10 ve Node.js gerekir.

```powershell
py -3.10 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-unified.txt

cd frontend
npm ci
```

## Yerel yapılandırma ve çalıştırma

Bu test deposu, kolay kurulum için yerel başlatma dosyasını da içerir. Anahtar
değeri yalnız test içindir; depo **özel (private)** tutulmalı ve bu anahtar
üretim ortamında kullanılmamalıdır. Gerekirse şablondan yeni bir yerel dosya da
oluşturabilirsiniz:

```powershell
Copy-Item .\scripts\start_backend_keyed.example.ps1 .\start_backend_keyed.local.ps1
.\start_backend_keyed.local.ps1
```

İkinci terminalde:

```powershell
cd frontend
npm run dev
```

## AI modelleri

Model ağırlıkları bu depoya eklenmez. Kullanacağınız model dosyalarını
`airport_ai/models/` klasörüne yerleştirin; beklenen dosyalar için
[model klasörü notlarına](airport_ai/models/README.md) bakın.

Mevcut model dosyaları eski altı sınıflı taksonomiyi kullanır:

`AIRCRAFT`, `AMBIENT`, `OTHER`, `SPEECH`, `TRAFFIC`, `WIND`

Bu sonuçlar doğrulama amaçlıdır. Güvenilir saha kullanımı için SVANTEK
kayıtlarıyla etiketli veri hazırlanmalı ve model yeniden eğitilmelidir.

## Bilinen sınırlama: SVL içindeki event sesi

SVL içindeki gömülü ses için mevcut Python extractor deneysel niteliktedir.
SvanPC++ ile üretilen WAV çıktısı farklı bir dönüşüm zinciri kullanabildiğinden,
bu sesler cızırtılı veya boğuk duyulabilir ve AI doğruluğunu düşürebilir.

Önerilen üretim yaklaşımı, SVAN setup'ında **Audio Recording: Wave** modunu
kullanmak ve cihazın oluşturduğu ayrı WAV dosyasını edge agent ile doğrudan
aktarmaktır. Böylece özel SVL ses decoder'ı devreden çıkar.

## Git politikası

Test kurulumu için yerel başlatma dosyası bilinçli olarak depoya eklenmiştir;
bu nedenle depo private kalmalıdır. Model ağırlıkları, kayıtlar, `.enc`
dosyaları, SQLite veritabanı, analiz çıktıları, `.venv` ve `node_modules`
eklenmez. Bağımlılıklar `requirements-unified.txt` ve `npm ci` ile yeniden
kurulur; model dosyaları ise yerel olarak `airport_ai/models/` içine konur.

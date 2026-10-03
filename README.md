# Baskı Maliyeti – Bambu Studio köprüsü

Bambu Studio'da her dilimlemeden sonra **baskı süresini ve filament gramını** Baskı Maliyeti Hesaplayıcı'ya kendiliğinden gönderen, tek dosyalık ve açık kaynak bir PowerShell betiği.

Kurmak isteğe bağlıdır. Hesaplayıcı köprüsüz de çalışır (STL, 3MF ya da ekran görüntüsüyle).

## Nasıl çalışır

Bambu Studio'nun kendi **İşlem Sonrası Komut Dosyaları** (Post-processing Scripts) özelliği, her dilimlemeden sonra bir programı çalıştırır ve ona G-code'un yolunu verir. Köprü:

1. G-code'un yalnızca ilk 64 KB'ını okur. Bambu Studio'nun ekranda gösterdiği özet orada yazar:
   ```
   ; model printing time: 17m 6s; total estimated time: 22m 14s
   ; total filament weight [g] : 7.21
   ```
2. Süreyi, gramı ve ayar adlarını (Bambu Studio bunları `SLIC3R_...` ortam değişkenleriyle verir) küçük bir JSON mesajı yapar.
3. Mesajı `https://ntfy.sh/<kanal-kodun>` adresine gönderir. Hesaplayıcı sayfası aynı kanalı dinler.

## Gönderdiği ve göndermediği

| Gönderir | Göndermez |
|---|---|
| Baskı süresi, filament gramı, katman sayısı | Model dosyası, G-code, proje dosyası |
| Yazıcı, nozul, işlem ve filament ayarının adı | Bilgisayar adı, kullanıcı adı, dosya yolları |
| Dosya yolunun 12 karakterlik özeti (aynı plakayı tanımak için) | |

G-code'u **değiştirmez**. Hata olursa (internet yok, dosya okunamadı) yine de Bambu Studio'ya "tamam" der, böylece **dilimleme asla durmaz**.

Örnek mesaj:
```json
{"v":1,"sn":1334,"model_sn":1026,"g":7.21,"gramlar":[7.21],"katman":39,
 "yazici":"Bambu Lab X2D","makine":"Bambu Lab X2D 0.4 nozzle","nozul":"0.4,0.4",
 "islem":"0.20mm Standard @BBL X2D","filament":"\"Bambu PLA Basic @BBL X2D 0.4 nozzle\"","ftur":"PLA",
 "plaka":"d4e80c079d86","proje":"3b0f3131f692","zaman":1790886591}
```

## Dosyanın aynı olduğunu kontrol etmek

Dosya herkes için **birebir aynıdır**. Sana özel kanal kodu dosyanın içinde değil, yalnızca adında (`baski-kopru-<kanal>.ps1`) ya da Bambu Studio'daki satırda yazar. Bu yüzden indirdiğin dosyanın SHA-256 parmak izi [`SHA256SUMS.txt`](SHA256SUMS.txt) ile aynı olmalıdır:

```powershell
Get-FileHash .\baski-kopru-*.ps1
```

VirusTotal sonucu: `https://www.virustotal.com/gui/file/<SHA256SUMS.txt içindeki değer>`

## Kurulum

### Kolay kurulum
1. Hesaplayıcıda **Bambu Studio'dan canlı al** → **Köprüyü indir**.
2. Dosyaya sağ tıkla → **PowerShell ile çalıştır** → **E**.
3. Bambu Studio'yu kapatıp aç.

Kolay kurulumun değiştirdikleri:
- `%LOCALAPPDATA%\BaskiKopru\`: dosyanın kopyası ve `kanal.txt`
- `%APPDATA%\BambuStudio\system\<marka>\process\fdm_process_common.json`: en üste tek satır `"post_process": [...]`. Bütün işlem ayarları bu dosyadan türediği için her yazıcıda ve işlem ayarında çalışır.
- Başlangıç klasöründe `BaskiKopru.lnk`: Bambu Studio kendi profillerini güncellerse satırı oturum açılışında sessizce yeniden ekler.

Kaldırmak: dosyayı yeniden çalıştır → **K**. Ayar dosyaları ilk haline döner, klasör ve kısayol silinir.

### Elle kurulum (hiçbir dosya değiştirilmez)
1. Dosyayı kalıcı bir klasöre koy, sağ tıkla → **Yol olarak kopyala**.
2. Hesaplayıcıdaki **Elle kurulum** sekmesine yolu yapıştır; sana Bambu Studio'ya yapıştıracağın satırı verir:
   ```
   "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\...\baski-kopru.ps1" -Kanal baski-xxxxxxxxxxxxxxxx
   ```
3. Bambu Studio → **İşlem** → **Gelişmiş** → **Diğer** → **İşlem Sonrası Komut Dosyaları** kutusuna yapıştır → **Kaydet**.

Kaldırmak: satırı silmen yeter.

## Neden `-ExecutionPolicy Bypass`?

Windows internetten gelen betikleri varsayılan olarak çalıştırmaz. `-ExecutionPolicy Bypass` bu izni bilgisayarın geneli için değil, **yalnızca o tek çalıştırma** için verir. `-WindowStyle Hidden`, her dilimlemede pencere açılmasın diye.

## ntfy.sh

[ntfy.sh](https://ntfy.sh) ücretsiz ve [açık kaynak](https://github.com/binwiederhier/ntfy) bir bildirim servisidir. Kanal kodu `baski-` ile başlayan 16 rastgele karakterdir. Kodu bilen herkes süre ve gramı görebilir, o yüzden paylaşma. Mesajlar ntfy.sh'de en fazla 12 saat tutulur.

## Gereksinimler

- Windows 10 / 11 (PowerShell 5.1 hazır gelir)
- Bambu Studio 2.x (İşlem Sonrası Komut Dosyaları her dilimlemede çalışır)

## Lisans

MIT. Ayrıntılar için [LICENSE](LICENSE).

---

### English summary

A single-file, open-source PowerShell script for Bambu Studio's *Post-processing Scripts* hook. After each slice it reads the summary lines at the top of the G-code (estimated time, filament weight), adds the preset names Bambu Studio passes as `SLIC3R_*` environment variables, and posts a small JSON message to a private random ntfy.sh topic that the Baskı Maliyeti cost calculator listens to. It never sends the model or G-code, never modifies the G-code, and always exits 0 so slicing is never blocked. The file is byte-identical for every user; verify it against `SHA256SUMS.txt`.

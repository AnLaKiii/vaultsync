# vaultsync — Obsidian Vault için Git Senkronizasyonu

Obsidian Sync yerine, kendi özel GitHub deponu kullanarak vault'unu senkronlayan basit
bir tesisat. **Uygulama değil, boru tesisatı:** vault zaten bir Markdown klasörü; onu
git ile senkronlar, geçmişi de git tutar.

- **Neden git?** Tam sürüm geçmişi istiyorsun. Geçmişe ihtiyacın yoksa Syncthing daha
  basit bir alternatif olurdu; burada bilinçli olarak git seçildi.
- **Geçmiş = `git log`.** Ayrı bir sürüm-geçmişi arayüzü yok (kapsam dışı).
- Notlar diskte düz Markdown olarak kalır. Telemetri yok, GitHub dışında hesap yok.

## Ne yapar

- **İzleyici (`watch`):** Son değişiklikten **30 sn** sonra (debounce) otomatik commit
  atar ve özel GitHub deposuna push eder.
- **Çekme:** Başlangıçta ve her **5 dakikada** bir `rebase` ile pull yapar.
- **Çatışmada asla veri kaybı yok:** gelen sürüm `<ad>.conflict-YYYY-AA-GG.md` olarak
  yazılır, seninki yerinde kalır, olay log'a düşer.
- **`vaultsync status`:** son push/pull zamanı ve bekleyen çatışma dosyalarını gösterir.
- **.gitignore:** `.obsidian/workspace*` ve `.trash/` yoksayılır; böylece cihaza özel
  durum makineler arasında gidip gelmez.

> **Not (WSL / ağ sürücüsü):** İzleyici dosya-sistemi olayları (inotify) yerine mtime
> yoklaması kullanır. Bu yüzden `/mnt/d/...` gibi WSL/drvfs yollarında da sorunsuz çalışır.

## Gereksinimler

`git` ve `bash`. Başka bağımlılık yok (Node/npm gerekmez).

## Kurulum

1. **Özel bir GitHub deposu oluştur** (ör. `obsidian-vault`) — **private** olduğundan emin ol.

2. **Scripti PATH'e koy:**
   ```bash
   mkdir -p ~/.local/bin
   ln -sf "$PWD/vaultsync" ~/.local/bin/vaultsync
   # ~/.local/bin PATH'te değilse ~/.bashrc'a ekle:
   #   export PATH="$HOME/.local/bin:$PATH"
   ```

3. **Ayar dosyasını oluştur:**
   ```bash
   cp vaultsync.conf.example vaultsync.conf
   # vaultsync.conf içinde VAULT_DIR ve REMOTE_URL'i düzenle
   ```

4. **Kimlik doğrulama** (iki seçenekten biri):
   - **Önerilen — sistem credential helper:**
     ```bash
     git config --global credential.helper store   # veya: manager / osxkeychain
     ```
     İlk push'ta kullanıcı adı + token bir kez sorulur, sonra hatırlanır.
   - **Alternatif — `.env` ile token:**
     ```bash
     cp .env.example .env      # GITHUB_USER ve GITHUB_TOKEN'ı doldur
     ```
     Script token'ı `GIT_ASKPASS` ile git'e verir; token `.git/config`'e **yazılmaz**.
     Token için ince yetkili (fine-grained) bir PAT ve sadece bu depoya `Contents: RW`
     izni yeterlidir.

5. **Depoyu hazırla ve ilk gönderimi yap:**
   ```bash
   vaultsync init
   vaultsync push
   ```

6. **Sürekli çalıştır** (aşağıdaki bir yöntemi seç).

### Otomatik çalıştırma

**Linux — systemd (kullanıcı servisi):**
```bash
mkdir -p ~/.config/systemd/user
cp systemd/vaultsync.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now vaultsync
# oturum kapalıyken de çalışsın (sunucu/başsız):
#   loginctl enable-linger "$USER"
journalctl --user -u vaultsync -f      # log akışı
```

**macOS — launchd:**
```bash
# plist içindeki KULLANICI_ADIN'i düzelt, sonra:
cp launchd/com.vaultsync.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.vaultsync.plist
```

**En basit — elle / nohup:**
```bash
nohup vaultsync watch >/dev/null 2>&1 &
```

## Günlük kullanım

```bash
vaultsync status     # son push/pull + çatışma dosyaları
vaultsync sync       # elle tek seferlik commit + push + pull
vaultsync pull       # elle çek
```

## İkinci makine kurulumu

1. Scripti kur (Kurulum adım 2) ve kimlik doğrulamayı ayarla (adım 4).
2. Depoyu klonla — bu makinede `init` yerine **klonlarsın**:
   ```bash
   git clone https://github.com/KULLANICI_ADIN/obsidian-vault.git "/hedef/yol/MyVault"
   ```
3. `vaultsync.conf` içinde `VAULT_DIR`'i bu yola, `REMOTE_URL`'i aynı depoya ayarla.
4. Bu klasörü Obsidian'da "Open folder as vault" ile aç.
5. Otomatik çalıştırmayı kur (systemd/launchd/nohup).

Artık iki makine de her 5 dakikada bir çekip değişikliklerini gönderir.

## Çatışma dosyası kuralı

İki makine aynı notu senkron olmadan değiştirdiğinde `pull` sırasında çatışma olur.
vaultsync **veriyi asla ezmez**:

- **Senin sürümün** dosyada **olduğu gibi kalır.**
- **Gelen (uzak) sürüm** aynı klasöre `<ad>.conflict-YYYY-AA-GG.md` olarak yazılır.
  Örn. `Gunluk/2026-08-10.md` çakışırsa → `Gunluk/2026-08-10.conflict-2026-08-10.md`.
- Olay log'a düşer ve `vaultsync status` çıktısında listelenir.

**Sen ne yaparsın:** İki dosyayı Obsidian'da yan yana aç, istediğini birleştir, sonra
`.conflict-...md` dosyasını sil. Silme işlemi de bir değişiklik olduğu için otomatik
commit'lenip diğer makinelere gider.

## Mobil (yalnızca doküman — uygulama yazılmadı)

Mobilde bir git istemcisini **aynı depoya** yönlendir; vaultsync mobilde çalışmaz:

- **iOS — Working Copy:** Depoyu klonla, klasörü Obsidian (iOS) vault'u olarak aç.
  Working Copy'de push/pull'u elle ya da otomasyonla (Shortcuts) tetikleyebilirsin.
- **Android — MGit** (veya Termux + git): Depoyu klonla, Obsidian (Android) vault'unu
  aynı klasöre yönlendir; senkronu MGit'ten elle yap.

Çatışma kuralı mobilde de aynıdır: gelen sürüm ayrı bir `.conflict-...` dosyası olur,
seninki korunur.

## Loglar ve durum

Log ve durum dosyaları vault'un **`.git/vaultsync/`** klasöründedir (senkronlanmaz):

- `vaultsync.log` — commit/push/pull ve çatışma kayıtları
- `last-push`, `last-pull` — zaman damgaları (`status` bunları okur)

## Kapsam dışı

- Şifreli-vault akışı.
- Sürüm-geçmişi arayüzü — geçmiş için `git log` kullan.
- Mobil uygulama — yukarıdaki hazır istemcileri kullan.

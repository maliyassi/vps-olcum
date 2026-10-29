# VPS-OLCUM: Yapay zekâ ajanı için VPS ölçüm talimatı

Bu bir program değil, terminal kullanabilen bir yapay zekâ ajanı (ör. Claude Code) için yazılmış bir talimat dosyasıdır. Kullanıcı bu dosyayı ajana verir ve "şu sunucumu ölç" der. Ajan aşağıdaki adımları sırayla uygular, sonunda bir **karne** (ilanda yazan ile ölçülenin karşılaştırması), bir **sonuç tablosu** ve bir **bileşik skor** verir.

- Tam ölçüm: yaklaşık **1 saat**. Hızlı mod: yaklaşık **10 dakika** (süre dökümü §3).
- Desteklenen sistem: **Ubuntu / Debian** (apt). RHEL ailesi (dnf) için farklar §12'de.
- Testler: işlemcinin yaşı, ikinci çekirdek, steal time ve gizli duraklama ölçümü, disk, yanıt süresi, yük testi, yarım saatlik dayanıklılık.

> **Ajan:** Bu dosyanın tamamını oku. §0'daki kurallar her adımda geçerlidir. Adımları atlama, sırasını değiştirme. Her adımın sonunda kullanıcıya kısa bir ara sonuç yaz.

---

## 0. Değişmez kurallar (ajan için)

1. **Sahiplik onayı olmadan hiçbir komut çalıştırma.** §1'deki soruya açık "evet" gelmezse dur.
2. **Kurulumdan önce onay al.** Kurulacak her paketin adını göster, "evet" bekle (§4). Silme işlemleri için de ayrı onay al (§8).
3. **Diskteki verilere dokunma.** Yazma testleri yalnız bu ölçüm için oluşturulan çalışma dizinindeki (`$W`) geçici dosyaya yapılır. Bir blok aygıta (`/dev/sda`, `/dev/vda`, `/dev/nvme0n1` …) yazan hiçbir komut çalıştırma: `dd of=/dev/...`, `mkfs`, `wipefs`, `fio --filename=/dev/...` (yazma) yasak. Ham aygıt yalnız `fio --readonly` ile ve kullanıcı onaylarsa okunur (§6, T5b).
4. **Port açma.** Güvenlik duvarına (`ufw`, `firewalld`, `iptables`, `nft`) ve sağlayıcının panelindeki kurallara dokunma. Test web sitesi yalnız `127.0.0.1` adresini dinler. Yük aracı aynı sunucuda çalışır ve isteği `127.0.0.1` üzerinden gönderir. Testten önce ve sonra dinlenen portların listesini al, karşılaştır (§4, §8).
5. **Şifreyi hiçbir yere yazma.** Kullanıcıdan şifre isteme. SSH anahtarı ile bağlan. Kullanıcı yine de şifre verirse onu dosyaya, komut satırına (`sshpass -p`, `echo şifre | sudo -S` gibi), log'a, rapora yazma ve sohbette tekrar etme. `sudo` şifre sorarsa şifreyi sen girme: kullanıcıdan o komutu kendi terminalinde çalıştırmasını iste.
6. **Mevcut servisleri durdurma, yeniden başlatma, ayarlarını değiştirme.** Test web sitesi sistemin nginx/php-fpm ayarlarını kullanmaz; `$W` içinden ayrı bir süreç olarak çalışır (§6, T7).
7. **Sonuç uydurma.** Bir test hata verirse o satıra "ölçülemedi" yaz ve nedenini (hata satırı) göster.
8. **Rapora IP adresi ve sunucu adı yazma.** Kullanıcı sonuçları paylaşabilir. Sunucuyu kullanıcının verdiği kısa bir etiketle an (ör. "sunucu-1").
9. Tüm geçici dosyalar tek bir dizinde durur: `W=$(mktemp -d /var/tmp/vps-olcum.XXXXXX)`. `/tmp` kullanma: bazı dağıtımlarda `/tmp` RAM üzerindedir (tmpfs), disk testi yanlış olur.

---

## 1. Sahiplik onayı

Kullanıcıya **ilk iş** şunu sor ve cevabı bekle:

> "Ölçeceğimiz sunucu size mi ait, ya da sahibinden bu testleri yapma izniniz var mı? Testler birkaç dakika boyunca işlemciyi ve diski tam yükte çalıştırır, sunucuda en çok yaklaşık 2 GB geçici dosya oluşturur ve sonra siler. Sunucuda canlı bir site varsa o süre içinde yavaşlar."

- Cevap açık bir "evet" değilse **dur**. Başka hiçbir adıma geçme.
- Sunucuda canlı bir site veya veritabanı çalışıyorsa bunu not et ve kullanıcıya ölçümü boş bir sunucuda ya da trafiğin az olduğu bir saatte yapmasını öner. Sonuçlar, sunucudaki diğer işlerden etkilenir.

---

## 2. Bağlantı, ilan bilgileri, mod

### 2a. Bağlantı

İki yol var. Kullanıcıya hangisini kullandığını sor:

- **Ajan kullanıcının bilgisayarında çalışıyor**, sunucuya SSH ile bağlanıyor. Kullanıcının SSH anahtarı zaten yüklü olmalı. Bağlantıyı şifresiz dene: `ssh -o BatchMode=yes <kullanıcı>@<sunucu> true`. Başarısızsa kullanıcıdan anahtarı kendisinin kurmasını iste (`ssh-copy-id <kullanıcı>@<sunucu>`; şifreyi o komutta kullanıcı kendisi girer). Bağlantı bilgisini (adres, kullanıcı) bir dosyaya yazma, yalnız komutlarda kullan.
- **Ajan doğrudan sunucuda çalışıyor.** Komutları yerelde çalıştır.

Yetki kontrolü: `id -u` 0 ise root'sun. Değilse `sudo -n true` dene. Şifresiz sudo yoksa §0 kural 5'e göre davran: yetki gereken komutları kullanıcıya ver, o çalıştırsın.

### 2b. İlan bilgileri (karne için)

Kullanıcıya sor (bilmediğini boş bırakabilir, o satır "— vaat yok" olur):

| Soru | Örnek cevap |
|---|---|
| İlanda kaç çekirdek (vCPU) yazıyor? | 2 |
| İşlemci için MHz/GHz yazıyor mu? | 2400 MHz |
| RAM | 4 GB |
| Disk alanı | 80 GB |
| Disk türü | NVMe |
| Hat hızı (port hızı) | 1 Gbit/s |
| Sunucu için kısa etiket (rapor adı) | sunucu-1 |

### 2c. Mod

Kullanıcıya "tam" (≈1 saat) mı "hızlı" (≈10 dakika) mı istediğini sor. Hızlı modun sonuçları ön izlemedir; skor tam modla karşılaştırılamaz (§3).

İsteğe bağlı iki ek testi de sor:
- **Ham disk okuma** (T5b): Diskin tamamına yayılan salt-okuma testi, disk türünü daha kesin gösterir. `fio --readonly` kullanır, veri yazmaz.
- **Önbelleksiz WordPress** (§11): Yük testini hafif PHP sayfası yerine gerçek bir WordPress ile yapar. Kurulum ≈10 dakika ekler, ek paket ve WordPress indirmesi gerektirir.

---

## 3. Süreler

| Adım | Tam mod | Hızlı mod |
|---|---|---|
| Sorular, paket kurulumu | ~3 dk | ~2 dk |
| T1 İşlemci yaşı | <10 sn | <10 sn |
| T2 İkinci çekirdek (sysbench 1 ve tüm iş parçacığı) | 2 × 20 sn | 2 × 5 sn |
| T3 Steal (boşta) + gizli duraklama | 30 + 20 + 20 sn | 15 + 5 + 5 sn |
| T4 RAM bant genişliği (ek bilgi) | 4 × 10 sn | atlanır |
| T5 Disk (2 GB dosya, 5 test × 30 sn + fsync) | ~4 dk | 256 MB, 5 × 10 sn ≈ 1 dk |
| T6 İnternet indirme (hat hızı) | ≤1 dk | ≤1 dk |
| T7 Yanıt süresi (site kurulumu + 2 × ölçüm) | ~1 dk + 2 × 30 sn | ~1 dk + 2 × 10 sn |
| T8 Yük testi (5 kademe) | en çok 5 × 3 dk = 15 dk | 5 × 10 sn |
| T9 Dayanıklılık + yük altında steal | 30 dk | 2 dk |
| Temizlik ve rapor | ~3 dk | ~1,5 dk |
| **Toplam** | **≈ 60 dk** | **≈ 10 dk** |

Yük testi bir kademede düşerse sonraki kademeler koşulmaz, toplam süre kısalır. Paket kurulumu ve disk hazırlığı sunucunun hızına göre değişir.

**Hızlı modda neler farklı:** test süreleri kısa, disk test dosyası 256 MB (önbellek etkisi artabilir), RAM testi yok, yük kademeleri 10 saniye (WordPress kriterindeki 3 dakika yerine), dayanıklılık 30 dakika yerine 2 dakika. Hızlı modun skoru "hızlı mod skoru" diye etiketlenir ve tam modun skoruyla yan yana konmaz.

---

## 4. Kurulum planı ve onay

### 4a. Çalışma dizini ve başlangıç durumu

Sunucuda (root veya sudo ile):

```bash
W=$(mktemp -d /var/tmp/vps-olcum.XXXXXX)
echo "$W"
mkdir -p "$W/out"
# Başlangıçta dinlenen portlar (temizlikte karşılaştırılacak)
ss -Hltnu | awk '{print $1, $5}' | sort -u > "$W/portlar_once.txt"
```

`$W` değerini not et. Sonraki tüm komutlarda aynı değeri kullan.

### 4b. Hangi paketler kurulacak

Gereken araçlar:

| Araç | Paket (apt) | Ne için |
|---|---|---|
| sysbench | `sysbench` | İşlemci ve RAM testleri |
| fio | `fio` | Disk testleri |
| nginx | `nginx` | Test web sitesi (yalnız 127.0.0.1) |
| php-fpm | `php-fpm` | Test sayfasını çalıştıran PHP |
| ab | `apache2-utils` | Yük aracı (yalnız istemci; Apache sunucusu kurulmaz) |
| python3, curl, taskset, ss | çoğunlukla kurulu (`python3`, `curl`, `util-linux`, `iproute2`) | Ölçüm betikleri, indirme testi |

Eksik olanları ve **bağımlılıklarla birlikte yeni kurulacak tüm paketleri** çıkar:

```bash
PAKETLER="sysbench fio nginx php-fpm apache2-utils"
for t in python3 curl taskset ss; do command -v $t >/dev/null || echo "EKSİK: $t"; done
apt-get update
apt-get -s install --no-install-recommends $PAKETLER | awk '/^Inst /{print $2}' > "$W/yeni-paketler.txt"
cat "$W/yeni-paketler.txt"
```

`python3`, `curl` vb. eksikse ilgili paketi (`python3`, `curl`, `util-linux`, `iproute2`) `PAKETLER` listesine ekle ve simülasyonu tekrarla.

Kullanıcıya göster: "Şu paketler kurulacak: … (liste). Testten sonra yalnız bu listeyi kaldırmayı önereceğim. Kurayım mı?" **"Evet" gelmeden kurma.**

`yeni-paketler.txt` boşsa her şey zaten kurulu demektir; temizlikte paket kaldırılmaz.

Not: `nginx` veya `php-fpm` sunucuda zaten kuruluysa (listeye girmezler) onlara dokunulmaz. Test sitesi kendi ayar dosyasıyla ayrı bir süreç olarak çalışır.

### 4c. Kurulum (onaydan sonra)

Debian/Ubuntu'da `nginx` paketi kurulunca normalde hemen başlar ve 80 numaralı portu dinler. Bunu engellemek için kurulum sırasında servislerin başlamasını geçici olarak kapat:

```bash
POLICY_BIZDEN=0
if [ ! -e /usr/sbin/policy-rc.d ]; then
  printf '#!/bin/sh\nexit 101\n' > /usr/sbin/policy-rc.d && chmod +x /usr/sbin/policy-rc.d && POLICY_BIZDEN=1
fi
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends $PAKETLER
[ "$POLICY_BIZDEN" = 1 ] && rm -f /usr/sbin/policy-rc.d
# Yeni kurulan nginx/php-fpm servisi yine de çalışıyorsa durdur (yalnız listede olanlar)
for s in $(grep -E '^(nginx|php[0-9.]*-fpm)$' "$W/yeni-paketler.txt"); do
  systemctl is-active -q "$s" && systemctl stop "$s"
done
ss -Hltnu | awk '{print $1, $5}' | sort -u | diff "$W/portlar_once.txt" - && echo "Yeni port yok"
```

`/usr/sbin/policy-rc.d` önceden varsa ona dokunma. Son satırda fark çıkarsa testlere geçme, kullanıcıya göster.

---

## 5. Betikleri yaz

Ek A–D'deki dört dosyayı `$W` içine yaz: `bilgi.py`, `gapprobe.py`, `olcum.sh`, `rapor.py`.

- Ajan sunucuda çalışıyorsa doğrudan `$W` içine yaz.
- Ajan kullanıcının bilgisayarındaysa dosyaları kendi geçici klasörüne yaz, `scp` ile sunucudaki `$W` içine kopyala.

Betiklerin içinde şifre, IP veya sunucu adı yoktur. Tüm test adımları şu biçimde çalışır (root veya `sudo env ...` ile):

```bash
W=<çalışma-dizini> MOD=tam bash "$W/olcum.sh" <adım>      # hızlı mod için MOD=hizli
```

---

## 6. Testler

Her testten sonra kullanıcıya bir iki satırlık ara sonuç yaz. Komut hata verirse §0 kural 7.

### T1. İşlemcinin yaşı

- **Ne ölçer:** Sanal sunucunun altındaki fiziksel işlemcinin modeli ve çıkış yılı.
- **Nasıl:** `/proc/cpuinfo` içindeki üretici, aile ve model numarası bir mimari tablosuyla eşlenir (Ek A).
- **Komut:** `olcum.sh bilgi`
- **Yorum:** Aynı "2 çekirdek" yazısının arkasında çok eski bir işlemci de olabilir, yeni bir işlemci de. Yeni nesil, aynı saat hızında daha çok iş yapar. Yaş tek başına hızı belirlemez; hız için T2'ye bakılır. Yaş = bu yıl − çıkış yılı. Tablo modeli tanımazsa "bilinmiyor" yaz; bazı sağlayıcılar model adını gizler.

### T2. İkinci çekirdek gerçekten iş yapıyor mu

- **Ne ölçer:** Sistem N çekirdek gösteriyor. İş N çekirdeğe bölününce gerçekten N kat iş çıkıyor mu?
- **Nasıl:** `sysbench cpu --cpu-max-prime=20000` önce 1 iş parçacığıyla, sonra N iş parçacığıyla. Ölçekleme = N parçacık ÷ 1 parçacık (olay/sn). İdeal: N kat. Verim = ölçekleme ÷ N.
- **Komut:** `olcum.sh cekirdek`
- **Yorum:** Verim %85 üstü: çekirdekler gerçek. %60 altı: ek çekirdekler kâğıt üstünde ya da aynı fiziksel kaynağı paylaşıyor. Tek vCPU'lu sunucuda bu test "—" olur.
- **Çapraz kontrol (verim düşükse, isteğe bağlı):** Tek araca güvenmemek için farklı bir işle tekrar et: `openssl speed -seconds 10 -multi 1 -evp sha256` ve `openssl speed -seconds 10 -multi N -evp sha256`. Toplam hız artmıyorsa sonuç doğrulanmış olur.

### T3. Steal time ve gizli duraklama

- **Steal time nedir:** Sanal işlemcinin çalışmak istediği halde hipervizörün (sanallaştırma katmanı) zamanı başka bir sanal makineye verdiği payı. `/proc/stat` içindeki `steal` sütunundan okunur.
- **Gizli duraklama nedir:** Bazı platformlar (ör. VMware) steal değerini sanal makinenin içine bildirmez. Orada steal %0 görünür ama bu "kayıp yok" değil, "kayıp görünmüyor" demektir. Bu yüzden ayrı bir ölçüm gerekir.
- **Nasıl:** `gapprobe.py` sıkı bir döngüde sürekli saati okur. İki okuma arasında saat **200 mikrosaniyeden** fazla ileri atlarsa, o arada işlemci durdurulmuş demektir. Atlamaların toplamı ÷ toplam süre = kayıp süre yüzdesi. Önce tek vCPU'da, sonra tüm vCPU'larda aynı anda çalışır; her kopya `taskset` ile bir vCPU'ya sabitlenir.
- **Komut:** `olcum.sh duraklama` (önce boşta steal örneklemesi, sonra gizli duraklama)
- **Yorum:**
  - Steal (boşta): %1 altı iyi, %1–5 dikkat, %5 üstü fiziksel sunucu aşırı yüklü. Yük altındaki steal T9'da ölçülür.
  - Gizli duraklama: %3 altı iyi, %15 üstü ciddi kayıp. Sanallaştırması düzgün bir KVM sunucuda tipik değer %1'in altındadır.
  - Tek vCPU'da kayıp düşük, tüm vCPU'larda yüksekse ve `loops_per_running_us` (çalıştığı anlardaki döngü hızı) iki durumda aynıysa: çekirdekler çalıştıkları anda yavaşlamıyor, hipervizör toplam süreyi kısıtlıyor. Bu, T2'deki düşük verimin sebebini açıklar.
  - Düzenli aralıklarla, aynı uzunlukta (yüzlerce ms) duraklamalar bir CPU limitine işaret edebilir. Ama limit mi, fiziksel sunucunun yoğunluğu mu olduğu içeriden kesin ayrılamaz; kesin cevap sağlayıcının hipervizör metriklerindedir. Rapora "işaret ediyor" diye yaz, "kanıtladı" deme.
  - Steal %0 tek başına olumlu bir sonuç olarak sunulmaz. Gizli duraklama sonucuyla birlikte yorumlanır.

### T4. RAM bant genişliği (yalnız tam mod, ek bilgi)

- **Komut:** `olcum.sh ram`
- `sysbench memory`, 1 MB bloklar, okuma ve yazma, 1 ve N iş parçacığı. N iş parçacığında hız artmak yerine düşüyorsa bu da T2/T3'teki CPU kısıtıyla tutarlıdır. Skora girmez.

### T5. Disk

- **Güvenlik:** Testler yalnız `$W/fio.testfile` dosyasında çalışır. Betik önce `$W`'nin RAM disk (tmpfs) olmadığını ve yeterli boş alan olduğunu (dosya boyutu + 1 GB) kontrol eder; yoksa durur. Test bitince dosya silinir.
- **Komut:** `olcum.sh disk`
- **Testler** (`fio`, `direct=1` ile işletim sistemi önbelleği atlanır, `libaio`):

| Test | Parametre | Ne anlatır |
|---|---|---|
| Sıralı okuma | 1M blok, kuyruk 16 | Büyük dosya okuma (yedek, büyük dosya servis etme) |
| Rastgele okuma | 4K blok, kuyruk 32 | Saniyede kaç küçük okuma (IOPS); veritabanının asıl yükü |
| Rastgele okuma, kuyruk 1 | 4K blok, kuyruk 1 | Tek bir okumanın gecikmesi; disk türü için ipucu |
| Sıralı yazma | 1M blok, kuyruk 16 | Büyük dosya yazma |
| Rastgele yazma | 4K blok, kuyruk 32 | Küçük yazma IOPS |
| fsync | 4K yazma, her yazmadan sonra fsync | Veritabanının "kalıcı kaydettim" dediği anın süresi |

- **Yorum:** IOPS değeri kuyruk derinliğine çok bağlıdır; başka araçların sonuçlarıyla yalnız aynı parametrelerde kıyaslanır. fsync ortalaması 1 ms altı iyi, 5 ms üstü yavaş. Çok düşük fsync süresi, depolamanın önündeki bir yazma önbelleğinden gelebilir; bu önbelleğin elektrik kesintisinde veriyi koruyup korumadığı içeriden bilinemez.

### T5b. Ham disk okuma (isteğe bağlı, yalnız onayla)

- **Ne ölçer:** Diskin tamamına yayılan tek tek 4 KB okumalarla disk türü. Mekanik disk her okumada okuma kafasını hareket ettirir: 5–10 ms. Flash (SSD/NVMe): 0,1 ms civarı.
- **Güvenlik:** `fio --readonly` bayrağı fio'nun yazma yapmasını engeller. Betikte bu bayrak sabittir.
- **Aygıtı bul:** `lsblk -o NAME,TYPE,SIZE,MOUNTPOINT` çıktısını kullanıcıya göster, kök dosya sisteminin bulunduğu diski (`TYPE=disk`) birlikte belirleyin. LVM veya RAID varsa altındaki diski seç.
- **Komut:** `olcum.sh disk-ham /dev/<disk>`
- Bu test atlanırsa disk türü T5'teki "rastgele okuma, kuyruk 1" gecikmesinden yorumlanır (dosya üzerinde olduğu için biraz daha yüksek çıkabilir).

### T6. İnternet indirme (hat hızı)

- **Ne ölçer:** Sunucunun internetten **indirme** hızı. Karnedeki "hat hızı" satırı için.
- **Adres:** Kullanıcıya büyük bir test dosyası adresi sor. Yoksa dağıtımın resmî sitesindeki güncel kurulum ISO'sunun bağlantısını kullan (ör. `releases.ubuntu.com` altındaki sürüm klasöründe `.iso` dosyası). Yalnız ilk 100 MB indirilir, diske yazılmaz (`/dev/null`).
- **Komut:** `olcum.sh ag <adres>`
- **Yorum:** Sonuç tek yönlüdür (gelen) ve karşı sunucunun hızıyla da sınırlıdır. Giden (upload) hız ölçülmez. Düşük bir sonuç için ikinci bir adresle tekrar et.

### T7. Yanıt süresi

- **Test sitesi:** nginx + php-fpm, `$W/web` içinden kendi ayar dosyalarıyla ayrı süreç olarak çalışır, **yalnız `127.0.0.1:18081`** adresini dinler. Sistemin `/etc/nginx` ayarlarına site eklenmez, mevcut servisler etkilenmez. php-fpm, paket varsayılanı olan `pm.max_children = 5` ile çalışır.
  - `/bench.php`: her istekte 20.000 tur sha256 hesaplayan sabit bir PHP sayfası (tipik bir dinamik sayfa gibi CPU harcar; tek istek birkaç on ms).
  - `/static.txt`: küçük sabit dosya (sunucu ek yükünün taban çizgisi).
- **Komutlar:**
  ```bash
  W=$W MOD=tam bash "$W/olcum.sh" web-baslat
  ss -Hltn | awk '{print $4}' | grep ':18081$'     # yalnız 127.0.0.1:18081 görünmeli
  W=$W MOD=tam bash "$W/olcum.sh" yanit
  ```
  `ss` çıktısında `0.0.0.0`, `*` veya `[::]` görünürse dur, `web-durdur` çalıştır, kullanıcıya göster. 18081 doluysa `PORT=<başka port>` ile tekrar başlat ve sonraki adımlarda aynı `PORT` değerini ver.
- **Ölçüm:** `ab` (Apache Bench) aynı sunucuda, tek kullanıcıyla `bench.php` ve `static.txt` için ortanca (p50) ve p99 yanıt süresi. p50 = isteklerin yarısı bu sürede ya da daha kısa sürede döndü. p99 = en yavaş %1 dışındaki her istek bu sürede döndü.
- **Yorum:** p99 ÷ p50 oranı 2'nin altındaysa yanıtlar tutarlı, 4'ün üstündeyse tutarsız. Ağ gecikmesi bu ölçümde yoktur (istek sunucunun içinden gelir); ziyaretçinin gördüğü süre buna ağ gecikmesi eklenmiş halidir.

### T8. Yük testi (aynı anda kaç kullanıcı)

- **Nasıl:** `ab` ile `bench.php`'ye 10 → 25 → 50 → 100 → 250 eş zamanlı kullanıcı. Tam modda her kademe 3 dakika, hızlı modda 10 saniye.
- **Düştü kriteri:** Hata oranı **%5'ten fazla** ya da **p95 yanıt süresi 3 saniyeden fazla**. Düştüğü kademede test durur. Sonuç = geçilen son kademe (hiçbiri geçilmezse 0).
- **Komut:** `olcum.sh yuk`, sonra `olcum.sh web-durdur`
- **Önemli sınır:** Yük aracı ölçülen sunucunun üzerinde çalışır ve onun işlemcisinden pay alır. Bu yüzden sonuç, yükün başka bir makineden geldiği bir testten biraz daha kötü çıkar; özellikle 1–2 çekirdekli sunucularda. Bunun karşılığında hiçbir port dışarıya açılmaz. Raporda bu notu yaz.

### T9. Yarım saatlik dayanıklılık (+ yük altında steal)

- **Ne ölçer:** İşlemci aralıksız tam yükte çalışınca hız zamanla düşüyor mu (kısma, ısınma, kota)?
- **Nasıl:** `sysbench cpu` N iş parçacığı, 1800 saniye (hızlı modda 120), 10 saniyede bir (hızlı modda 5) kayıt. Aynı anda steal örneklenir. İlk %10'luk dilimin ortalaması ile son %10'luk dilimin ortalaması kıyaslanır.
- **Komut:** SSH koparsa test sürsün diye arka planda başlat:
  ```bash
  nohup env W=$W MOD=tam bash "$W/olcum.sh" dayaniklilik > "$W/out/dayaniklilik.log" 2>&1 < /dev/null &
  ```
  Bitişi `test -f "$W/out/dayaniklilik.bitti"` ile birkaç dakikada bir kontrol et. Beklerken kullanıcıya önceki testlerin ara sonuçlarını ver.
- **Yorum:** ±%5 içindeki değişim ölçüm gürültüsüdür. %15 üstü düşüş kısma ya da komşu baskısıdır. En kötü anlık düşüşü ve dalgalanmayı (CoV, standart sapma ÷ ortalama) da yaz. Bu test zaten var olan gücün kararlı olup olmadığını gösterir; T2'deki bir verim sorununu ortadan kaldırmaz.

---

## 7. Sonuçları topla

```bash
python3 "$W/rapor.py" "$W/out"
```

Çıktı: Markdown sonuç tablosu, skor tablosu ve `$W/out/ozet.json`. Karneyi (§9) bu çıktı ile §2b'deki ilan bilgilerinden sen doldur. Ham dosyalar silinmeden önce kullanıcıya sor: "Ham sonuçları (`$W/out`, birkaç yüz KB) bilgisayarınıza kopyalayayım mı?" Evet ise `scp -r` ile kullanıcının seçtiği yerel klasöre al.

---

## 8. Temizlik

Sırayla yap. Her silmeden önce ne silineceğini göster. `$W` en sonda silinir, çünkü paket listesi ve başlangıç port listesi onun içindedir.

1. **Test süreçleri:** `olcum.sh web-durdur`. Kalan süreç var mı: `pgrep -af "$W"` (çıktı boş olmalı; varsa PID'leri göster, onayla `kill`).
2. **Paketler:** `cat "$W/yeni-paketler.txt"` çıktısını göster ve sor: "Ölçüm için kurulan bu paketleri kaldırayım mı?"
   - Evet: `apt-get purge -y $(cat "$W/yeni-paketler.txt")`. Sonra `apt-get -s autoremove` çıktısını göster; ayrı bir onayla `apt-get autoremove --purge -y`. Listede olmayan hiçbir paketi kaldırma.
   - Hayır: Yeni kurulan nginx ve php-fpm servislerini kapat ki sunucu yeniden başlayınca 80 portunu açmasın: `systemctl disable --now nginx php<sürüm>-fpm` (yalnız listede olanlar).
3. **Anahtarlar ve kalıntılar:** Bu talimat kullanıcı hesabı oluşturmaz. Kullanıcı ölçüm için ayrı bir SSH anahtarı eklediyse, onayıyla `~/.ssh/authorized_keys` içinden yalnız o satırı kaldır (satırı önce göster). `/usr/sbin/policy-rc.d` dosyasını biz oluşturduysak ve hâlâ duruyorsa sil.
4. **Port kontrolü:** `ss -Hltnu | awk '{print $1, $5}' | sort -u | diff "$W/portlar_once.txt" -` çıktısı boş olmalı. Fark varsa kullanıcıya göster.
5. **Geçici dosyalar:** Ham sonuçlar kopyalandıktan (ya da kullanıcı istemedikten) sonra `case "$W" in /var/tmp/vps-olcum.*) rm -rf "$W";; esac`. Bu yol kalıbı dışında hiçbir şey silme. Ajan kullanıcının bilgisayarında çalıştıysa kendi geçici klasöründeki betik kopyalarını da sil.
6. Kullanıcıya özet ver: ne silindi, ne bırakıldı (ör. "paketleri bıraktınız, servisler kapalı").

---

## 9. Rapor: karne, sonuç tablosu, skor

Raporu sohbete yaz. Kullanıcı isterse `.md` dosyası olarak kaydet. IP ve sunucu adı yazma (§0 kural 8). Raporun başına şunu koy:

> Sunucu: <etiket> · Mod: tam/hızlı · Ölçüm başlangıcı: <tarih saat> · Yöntem: VPS-OLCUM

### 9a. Karne (vaat ve gerçek)

| Satır | İlanda | Ölçülen | Sonuç |
|---|---|---|---|
| Çekirdek sayısı | … | `nproc` | ✓ / ✗ |
| İkinci çekirdek verimi | N çekirdek = N kat iş | ölçekleme (ör. 1,00x), verim % | ✓ / ~ / ✗ |
| CPU vaadi | … MHz | hak edilen ve gelen çekirdek | ✓ / ~ / ✗ |
| RAM | … GB | `MemTotal` | ✓ / ~ / ✗ |
| Disk alanı | … GB | disk ve dosya sistemi boyutu | ✓ / ~ / ✗ |
| Disk türü | NVMe | gecikmeye göre flash / mekanik | ✓ / ✗ |
| Hat hızı | … Mbit/s | indirme Mbit/s | ✓ / ~ |
| Güncel sistem | — | dağıtım ve sürüm | ✓ / ~ / ✗ |

Sonuç kuralları (ilanda bilgi yoksa "— vaat yok"):

- **Çekirdek sayısı:** `nproc` ≥ ilan → ✓, az → ✗.
- **İkinci çekirdek verimi:** verim ≥ %85 → ✓, %60–85 → ~, ≤ %60 → ✗. Tek vCPU → "—".
- **CPU vaadi:** İlanda MHz yazıyorsa: hak edilen çekirdek = ilan MHz ÷ işlemcinin taban frekansı (MHz). Taban frekans model adındaki "@ x.xx GHz" değeridir (`bilgi.json` → `taban_ghz`); model adında yoksa `lscpu` çıktısındaki değeri kullan ve hangi değeri kullandığını yaz. Gelen çekirdek = T2 ölçeklemesi. Oran = gelen ÷ hak edilen. ≥ %90 → ✓, %70–90 → ~, altı → ✗. Bu eşikler bu dosyanın seçimidir; raporda oranın kendisini de yaz. İlanda yalnız çekirdek sayısı yazıyorsa bu satır "—" (verim satırı zaten ölçüyor).
- **RAM:** `MemTotal` ilandaki değerin %90'ı ve üstü ama altında → ~ (çekirdeğin ayırdığı pay normaldir), ilan ve üstü → ✓, %90 altı → ✗.
- **Disk alanı:** Disk (`lsblk`) veya kök dosya sistemi boyutu ilan ve üstü → ✓, %90 ve üstü → ~, altı → ✗. İki değeri de yaz.
- **Disk türü:** 4K kuyruk 1 okuma gecikmesi (tercihen ham disk) 1 ms altı → flash, 3 ms üstü → mekanik. İlan SSD/NVMe diyor ve flash çıktıysa ✓. NVMe ile SATA SSD ayrımı içeriden bu testle kesin yapılamaz; raporda "flash olduğu doğrulandı" de, "NVMe doğrulandı" deme.
- **Hat hızı:** İndirme ilanın %80'i ve üstü → ✓, altı → ~ ("tek yönlü, tek kaynaklı ölçüm; ✗ demek için yeterli değil").
- **Güncel sistem:** Dağıtımın bu sürümü üreticisinin standart desteği içindeyse ✓, desteği bitmişse ✗, emin değilsen ~ ve sürüm adını yaz.

Altına **✗ sayısını** yaz.

### 9b. Sonuç tablosu

`rapor.py` çıktısındaki tabloyu kullan. En az şunlar olmalı: işlemci modeli, mimari ve yıl, sanallaştırma türü, tek ve çok iş parçacığı olay/sn, ölçekleme ve verim, steal (boşta / yük altında), gizli duraklama (tek / tüm vCPU), RAM bant genişliği (varsa), disk okuma ve yazma tabloları, fsync, indirme hızı, yanıt süresi (p50 / p99), yük kademeleri tablosu (istek/sn, p50, p95, hata %) ve geçilen son kademe, 30 dk düşüşü, en kötü anlık düşüş, dalgalanma.

Her önemli satırın yanına bir cümlelik yorum ekle (§6'daki eşiklerle).

### 9c. Bileşik skor

`rapor.py` skor tablosunu da basar. Kuralları §10'da. Skorun altına mutlaka şu notu yaz:

> Bu skor yaklaşık bir kıyas içindir. Referans değerler başka bir ölçüm düzeninden geliyor: web testleri orada önbelleksiz WordPress ile ve dışarıdaki bir makineden yapıldı; burada (WordPress seçilmediyse) hafif bir PHP sayfası ile ve sunucunun kendi içinden yapıldı. fio ve sysbench parametreleri aynı tutulmaya çalışıldı ama birebir aynı olduğu garanti değil.

---

## 10. Skor (v2)

**6 test, her biri 0–100 puan. Skor = 6 puanın ortalaması (eşit ağırlık).** Ölçek sabittir: referans değer 100 puandır, onu geçen de 100 alır. Böylece yeni bir sunucu eklemek eskilerin puanını değiştirmez.

| Test | Kullanılan değer | Puan | 100 puanın karşılığı |
|---|---|---|---|
| İşlemci gücü | Tüm çekirdekler birlikte, sysbench olay/sn (asal 20.000) | değer ÷ 1.439 × 100 | 1.439 olay/sn |
| Büyük dosya okuma | Sıralı okuma MB/s | değer ÷ 2.600 × 100 | 2.600 MB/s |
| Küçük parça okuma | 4K rastgele okuma IOPS (kuyruk 32) | değer ÷ 103.531 × 100 | 103.531 IOPS |
| 30 dk dayanıklılık | Yarım saatte yavaşlama % | 100 − yavaşlama % (hızlanma 100 sayılır) | Hiç yavaşlamamak |
| Web yanıt | 10 kullanıcıda ortanca (p50) yanıt, ms | 43 ÷ değer × 100 | 43 ms |
| Web yük | Geçilen son kullanıcı kademesi | kademe ÷ 100 × 100 | 100 kademesini geçmek |

Formüle girmeyenler: steal ve gizli duraklama (zararı zaten işlemci gücü puanına yansır, ikinci kez saymak çift ceza olur), ağ hızı, disk yazma, fsync, RAM ve karne. Karne ayrı bir başlıktır: "ilan ne vaat ediyor" ile "ne kadar iyi" farklı sorulardır.

Bir test ölçülemediyse `rapor.py` skoru aralık olarak verir: eksik test 0 alırsa / 100 alırsa.

---

## 11. İsteğe bağlı: önbelleksiz WordPress yük testi

Kullanıcı §2c'de istediyse T7–T8'i hafif PHP sayfası yerine WordPress ile yap. Ek kurallar:

- **Ek paketler:** `mariadb-server php-mysql` (ve WordPress'in istediği `php-xml php-mbstring php-curl php-gd php-zip` gibi uzantılar). §4b'deki simülasyonla tam listeyi çıkar, onay al, `policy-rc.d` yöntemiyle kur.
- **WordPress dosyası:** Yalnız resmî siteden (`wordpress.org/latest.tar.gz`) ve kullanıcıya dosya adını ve kaynağını söyleyip onay aldıktan sonra indir. `$W/web/kok` içine aç.
- **Veritabanı:** MariaDB'yi de `$W` içinde ayrı bir veri dizini ve yalnız `127.0.0.1` (ya da yalnız soket) dinleyen ayrı bir süreç olarak başlat; sistemin MariaDB'sine dokunma. Veritabanı şifresini rastgele üret; bu şifre yalnız bu test sitesine aittir, `$W` içindeki `wp-config.php` dosyasında durur ve temizlikte `$W` ile birlikte silinir. Kullanıcının hiçbir şifresi kullanılmaz.
- **Kurulum:** Varsayılan tema, örnek içerik, **önbellek eklentisi yok**, PHP opcache paket varsayılanında. Ölçülen sayfa ana sayfadır. Test sitesinin `nginx.conf` dosyasındaki `server` bloğuna `index index.php;` ve `location / { try_files $uri $uri/ /index.php?$args; }` ekle.
- **Ölçüm:** 10 kullanıcıda ortanca yanıt (skorun "web yanıt" satırı), sonra **10 → 25 → 50 → 100 → 250** kullanıcı, **her kademe 3 dakika**, düştü kriteri **hata > %5 veya p95 > 3 sn**. `olcum.sh yuk` adımındaki döngüyü `bench.php` yerine `/` ile çalıştır (`ab` çıktı dosyalarını `ab_c<N>.txt` adıyla aynı yere yaz, `rapor.py` aynen okur).
- **Süre:** Kurulum ≈10 dk, yük en çok 15 dk. Tam mod toplamı ≈ 70 dk olur.
- **Raporda:** "Web testi: önbelleksiz WordPress, yük aracı aynı sunucuda" yaz.

---

## 12. RHEL ailesi (dnf) için farklar

- Paketler: `sysbench fio nginx php-fpm httpd-tools` (`ab`, `httpd-tools` içindedir). `sysbench` çoğunlukla EPEL deposundadır. EPEL eklemek kalıcı bir depo değişikliğidir: ayrı onay al, temizlikte kaldırmayı öner.
- Kurulacak paket listesi: `dnf install --assumeno <paketler>` çıktısı. Kurulumdan sonra işlem numarasını `dnf history` ile not et; temizlikte `dnf history undo <numara>` (önce göster, onay al).
- RHEL'de servisler kurulumla başlamaz; `policy-rc.d` gerekmez. Yine de kurulumdan sonra port listesini karşılaştır.
- `olcum.sh` php-fpm'i `/usr/sbin/php-fpm*` içinde bulur, `www-data` kullanıcısı yoksa `nobody` kullanır.

---

## 13. Sınırlılıklar ve iki rica

Raporun sonuna şu bölümü aynen ekle:

> **Bu ölçümün sınırları**
> - Tek bir ölçüm, o günün sonucudur. Sanal sunucuda komşu yükü değişir; aynı sunucunun tek çekirdek hızı iki ölçüm arasında belirgin oynayabilir. Kesin bir yargı için farklı gün ve saatlerde tekrar edin.
> - Gizli duraklamanın kaynağı (CPU limiti mi, fiziksel sunucunun yoğunluğu mu) içeriden kesin ayrılamaz.
> - Yük aracı ölçülen sunucunun içinde çalıştı; dışarıdan yapılan testlerden biraz daha kötü sonuç verir. Ağ gecikmesi yanıt sürelerine dahil değildir.
> - İnternet hızı yalnız indirme yönünde ve tek kaynaktan ölçüldü.
> - Disk yazma önbelleğinin elektrik kesintisindeki davranışı, çalışma süresi (uptime) ve destek hızı ölçülmedi.
> - Skor yaklaşık bir kıyastır (§9c).
>
> **İki rica**
> 1. Ölçtüğünüz sunucu sizin olsun.
> 2. Sonuçlarınızı paylaşırken yöntemi de paylaşın.

---

## Ek A: `bilgi.py`

```python
#!/usr/bin/env python3
"""İşlemci yaşı ve sistem bilgisi. Kullanım: python3 bilgi.py > bilgi.json"""
import json
import os
import re
import subprocess

# Intel family 6 model numarası -> (mimari, çıkış yılı)
INTEL = {
    0x1A: ("Nehalem", 2008), 0x2C: ("Westmere", 2010), 0x2F: ("Westmere-EX", 2011),
    0x2A: ("Sandy Bridge", 2011), 0x2D: ("Sandy Bridge-EP", 2012),
    0x3A: ("Ivy Bridge", 2012), 0x3E: ("Ivy Bridge-EP", 2013),
    0x3C: ("Haswell", 2013), 0x3F: ("Haswell-EP", 2014),
    0x3D: ("Broadwell", 2015), 0x4F: ("Broadwell-EP", 2016), 0x56: ("Broadwell-DE", 2015),
    0x55: ("Skylake-SP", 2017), 0x5E: ("Skylake", 2015), 0x8E: ("Kaby/Coffee Lake", 2017),
    0x9E: ("Kaby/Coffee Lake", 2017), 0xA5: ("Comet Lake", 2020), 0xA7: ("Rocket Lake", 2021),
    0x6A: ("Ice Lake-SP", 2021), 0x6C: ("Ice Lake-D", 2021),
    0x97: ("Alder Lake", 2021), 0x9A: ("Alder Lake", 2021),
    0xB7: ("Raptor Lake", 2022), 0xBF: ("Raptor Lake", 2023),
    0x8F: ("Sapphire Rapids", 2023), 0xCF: ("Emerald Rapids", 2023),
    0xAD: ("Granite Rapids", 2024), 0xAF: ("Sierra Forest", 2024),
}


def alan(metin, anahtar):
    m = re.search(rf"^{anahtar}\s*:\s*(.*)$", metin, re.M)
    return m.group(1).strip() if m else ""


def sayi(s):
    try:
        return int(s)
    except ValueError:
        return None


def yas(vendor, fam, model, flags, ad):
    if vendor == "GenuineIntel" and fam == 6 and model in INTEL:
        mimari, yil = INTEL[model]
        if model == 0x55 and "avx512_vnni" in flags:
            mimari = "Cascade Lake" + (" / Cooper Lake" if "avx512_bf16" in flags else "")
            yil = 2019
        return mimari, yil
    if vendor == "AuthenticAMD" and model is not None:
        if fam == 23:
            return ("Zen 2 (Rome)", 2019) if model >= 0x30 else ("Zen/Zen+ (Naples)", 2017)
        if fam == 25:
            if model < 0x10 or 0x20 <= model < 0x60:
                return "Zen 3 (Milan/Vermeer)", 2021
            if model >= 0xA0:
                return "Zen 4c (Bergamo)", 2023
            return "Zen 4 (Genoa/Raphael)", 2022
        if fam == 26:
            return "Zen 5 (Turin)", 2024
    m = re.search(r"EPYC (\d)\d\d(\d)", ad)
    if m:
        nesil = {"1": ("Naples", 2017), "2": ("Rome", 2019), "3": ("Milan", 2021),
                 "4": ("Genoa", 2022), "5": ("Turin", 2024)}.get(m.group(2))
        if nesil:
            return nesil
    m = re.search(r"\((\w+lake|\w+well|\w+bridge|\w+ Rapids)", ad, re.I)
    if m:
        return m.group(1), None
    return "bilinmiyor", None


ci = open("/proc/cpuinfo").read()
vendor, ad = alan(ci, "vendor_id"), alan(ci, "model name")
fam, model = sayi(alan(ci, "cpu family")), sayi(alan(ci, "model"))
flags = alan(ci, "flags").split()
mimari, yil = yas(vendor, fam, model, flags, ad)
taban = re.search(r"@\s*([\d.]+)\s*GHz", ad)
try:
    virt = subprocess.run(["systemd-detect-virt"], capture_output=True, text=True).stdout.strip()
except FileNotFoundError:
    virt = ""
mem = re.search(r"MemTotal:\s+(\d+)", open("/proc/meminfo").read())
st = os.statvfs("/")
osr = {}
if os.path.exists("/etc/os-release"):
    for satir in open("/etc/os-release"):
        if "=" in satir:
            k, v = satir.rstrip("\n").split("=", 1)
            osr[k] = v.strip('"')

print(json.dumps({
    "model_adi": ad, "uretici": vendor, "aile": fam, "model": model,
    "mimari": mimari, "cikis_yili": yil,
    "taban_ghz": float(taban.group(1)) if taban else None,
    "cpu_mhz_anlik": alan(ci, "cpu MHz") or None,
    "vcpu": len(os.sched_getaffinity(0)),
    "ram_mb": int(mem.group(1)) // 1024 if mem else None,
    "kok_dosya_sistemi_gb": round(st.f_blocks * st.f_frsize / 1e9, 1),
    "sanallastirma": virt or "bilinmiyor",
    "cekirdek_surumu": os.uname().release,
    "sistem": osr.get("PRETTY_NAME", "bilinmiyor"),
}, ensure_ascii=False, indent=2))
```

## Ek B: `gapprobe.py`

```python
#!/usr/bin/env python3
"""Gizli duraklama ölçümü: sıkı döngüde saat okur, vCPU'nun hipervizör tarafından
durdurulduğu boşlukları (eşik üstü atlamalar) toplar. Platform steal bildirmese de
kaybedilen duvar saati süresi buradan görülür.

Kullanım: gapprobe.py <süre-sn> [eşik-µs, varsayılan 200]
"""
import json
import statistics
import sys
import time

dur_ns = int(float(sys.argv[1]) * 1e9)
thr_ns = int(float(sys.argv[2] if len(sys.argv) > 2 else 200) * 1000)

pc = time.perf_counter_ns
start = last = pc()
gaps, iters = [], 0
while True:
    now = pc()
    d = now - last
    if d > thr_ns:
        gaps.append(d)
    last = now
    iters += 1
    if now - start > dur_ns:
        break

wall = last - start
lost = sum(gaps)
print(json.dumps({
    "wall_s": round(wall / 1e9, 2),
    "lost_pct": round(100 * lost / wall, 1),
    "gaps": len(gaps),
    "gap_median_ms": round(statistics.median(gaps) / 1e6, 2) if gaps else 0,
    "gap_max_ms": round(max(gaps) / 1e6, 2) if gaps else 0,
    "loops_per_running_us": round(iters / ((wall - lost) / 1000), 2),
}))
```

## Ek C: `olcum.sh`

```bash
#!/usr/bin/env bash
# VPS-OLCUM test adımları. Ölçülen sunucuda root olarak (ya da sudo env ... ile) çalışır.
# Kullanım: W=<çalışma-dizini> MOD=tam|hizli [PORT=18081] bash olcum.sh <adım> [argüman]
# Adımlar: bilgi | cekirdek | duraklama | ram | disk | disk-ham <aygıt> | ag <adres>
#          web-baslat | yanit | yuk | web-durdur | dayaniklilik
# Yazdığı her şey $W içindedir. Blok aygıta yazmaz, port açmaz, güvenlik duvarına dokunmaz.
set -uo pipefail
: "${W:?W (çalışma dizini) ayarlanmalı}"
ADIM=${1:?adım}
MOD=${MOD:-tam}
PORT=${PORT:-18081}
URL="http://127.0.0.1:$PORT"
OUT=$W/out
mkdir -p "$OUT"
N=$(nproc)
CPUS=$(python3 -c 'import os; print(" ".join(map(str, sorted(os.sched_getaffinity(0)))))')
ILK_CPU=${CPUS%% *}

if [ "$MOD" = hizli ]; then
  CEK=5;  BOSTA=15; DISK=10; BOYUT=256M; FSYNC=16M; C1=10; KADEME=10;  DAY=120;  DAYARA=5
else
  CEK=20; BOSTA=30; DISK=30; BOYUT=2G;   FSYNC=64M; C1=30; KADEME=180; DAY=1800; DAYARA=10
fi

log() { echo "[$(date +%H:%M:%S)] $*" >&2; }

# /proc/stat toplam CPU sayaçları: user nice system idle iowait irq softirq steal
cpu_sayac() { awk '/^cpu /{print $2,$3,$4,$5,$6,$7,$8,$9}' /proc/stat; }

# Her ARALIK saniyede CPU dağılımını CSV'ye yazar; SURE dolunca çıkar.
steal_ornekle() {
  local sure=$1 aralik=$2 csv=$3 t=0 once simdi
  echo "t,user_pct,system_pct,idle_pct,iowait_pct,steal_pct" > "$csv"
  once=$(cpu_sayac)
  while [ "$t" -lt "$sure" ]; do
    sleep "$aralik"; t=$((t + aralik))
    simdi=$(cpu_sayac)
    awk -v t="$t" -v p="$once" -v c="$simdi" 'BEGIN{
      split(p,a," "); split(c,b," "); tot=0
      for(i=1;i<=8;i++){d[i]=b[i]-a[i]; tot+=d[i]}
      if(tot==0) tot=1
      printf "%d,%.2f,%.2f,%.2f,%.2f,%.2f\n", t, 100*(d[1]+d[2])/tot, 100*(d[3]+d[6]+d[7])/tot, 100*d[4]/tot, 100*d[5]/tot, 100*d[8]/tot
    }' >> "$csv"
    once=$simdi
  done
}

# ab çıktısı kademeyi geçti mi: hata <= %5 ve p95 <= 3000 ms
ab_gecti() {
  awk '/^Complete requests:/{c=$3} /^Failed requests:/{f=$3} /^Non-2xx responses:/{n=$3}
       /^ *95%/{p=$2}
       END{ if (c+0 == 0 || p == "") exit 1; e = 100*(f+n)/c; exit !(e <= 5 && p+0 <= 3000) }' "$1"
}

case "$ADIM" in
bilgi)
  python3 "$W/bilgi.py" > "$OUT/bilgi.json"
  lscpu > "$OUT/lscpu.txt" 2>&1
  free -m > "$OUT/free.txt"
  df -h / > "$OUT/df.txt"
  lsblk -d -o NAME,SIZE,ROTA,TYPE,MODEL > "$OUT/lsblk.txt" 2>&1
  cat "$OUT/bilgi.json"
  ;;

cekirdek)
  for th in 1 "$N"; do
    log "sysbench cpu, $th iş parçacığı, ${CEK} sn"
    sysbench cpu --cpu-max-prime=20000 --threads="$th" --time="$CEK" run > "$OUT/cpu_t$th.txt"
    grep "events per second" "$OUT/cpu_t$th.txt"
  done
  ;;

duraklama)
  log "Boşta steal örneklemesi (${BOSTA} sn)"
  steal_ornekle "$BOSTA" 5 "$OUT/steal_bosta.csv"
  log "Gizli duraklama: tek vCPU (${CEK} sn)"
  taskset -c "$ILK_CPU" python3 "$W/gapprobe.py" "$CEK" > "$OUT/gap_tek.json"
  log "Gizli duraklama: tüm vCPU'lar aynı anda (${CEK} sn)"
  for i in $CPUS; do
    taskset -c "$i" python3 "$W/gapprobe.py" "$CEK" > "$OUT/gap_hepsi_$i.json" &
  done
  wait
  cat "$OUT"/gap_*.json
  ;;

ram)
  for op in read write; do
    for th in 1 "$N"; do
      log "sysbench memory $op, $th iş parçacığı, 10 sn"
      sysbench memory --memory-block-size=1M --memory-total-size=1000G --memory-oper="$op" \
        --threads="$th" --time=10 run > "$OUT/mem_${op}_t$th.txt"
    done
  done
  ;;

disk)
  F=$W/fio.testfile
  fstype=$(df -PT "$W" | awk 'NR==2{print $2}')
  case "$fstype" in tmpfs|ramfs) log "HATA: $W bir RAM disk ($fstype) üzerinde; disk ölçülemez"; exit 2 ;; esac
  gerek=$(( $(numfmt --from=iec "$BOYUT") / 1048576 + 1024 ))
  bos=$(df -Pm "$W" | awk 'NR==2{print $4}')
  [ "$bos" -gt "$gerek" ] || { log "HATA: boş alan ${bos} MB, gereken ${gerek} MB"; exit 2; }
  log "Test dosyası hazırlanıyor ($BOYUT, yalnız $F)"
  fio --name=hazirlik --filename="$F" --size="$BOYUT" --rw=write --bs=1M --direct=1 \
    --ioengine=libaio --iodepth=8 --output=/dev/null
  ortak=(--filename="$F" --size="$BOYUT" --direct=1 --ioengine=libaio --time_based
         --runtime="$DISK" --group_reporting --output-format=json)
  log "Sıralı okuma 1M QD16";        fio --name=seqread      --rw=read      --bs=1M --iodepth=16 "${ortak[@]}" > "$OUT/fio_seqread.json"
  log "Rastgele okuma 4K QD32";      fio --name=randread     --rw=randread  --bs=4k --iodepth=32 "${ortak[@]}" > "$OUT/fio_randread.json"
  log "Rastgele okuma 4K QD1";       fio --name=randread_qd1 --rw=randread  --bs=4k --iodepth=1  "${ortak[@]}" > "$OUT/fio_randread_qd1.json"
  log "Sıralı yazma 1M QD16";        fio --name=seqwrite     --rw=write     --bs=1M --iodepth=16 "${ortak[@]}" > "$OUT/fio_seqwrite.json"
  log "Rastgele yazma 4K QD32";      fio --name=randwrite    --rw=randwrite --bs=4k --iodepth=32 "${ortak[@]}" > "$OUT/fio_randwrite.json"
  log "fsync gecikmesi ($FSYNC)"
  fio --name=fsync --rw=write --bs=4k --size="$FSYNC" --fsync=1 --ioengine=sync \
    --filename="$F" --output-format=json > "$OUT/fio_fsync.json"
  rm -f "$F"
  log "Test dosyası silindi"
  ;;

disk-ham)
  DEV=${2:?aygıt, ör. /dev/sda}
  [ -b "$DEV" ] || { log "HATA: $DEV bir blok aygıt değil"; exit 2; }
  log "Ham disk salt-okuma testi (--readonly): $DEV, 4K QD1, ${DISK} sn"
  fio --name=ham_qd1 --filename="$DEV" --readonly --rw=randread --bs=4k --direct=1 \
    --ioengine=libaio --iodepth=1 --time_based --runtime="$DISK" --output-format=json > "$OUT/fio_ham_qd1.json"
  ;;

ag)
  ADRES=${2:?indirme adresi}
  log "İndirme testi: ilk 100 MB, diske yazılmaz"
  curl -s -o /dev/null -m 60 -r 0-104857599 \
    -w '%{speed_download} %{size_download} %{time_starttransfer}\n' "$ADRES" > "$OUT/ag.txt" || true
  cat "$OUT/ag.txt"
  ;;

web-baslat)
  if ss -Hltn | awk '{print $4}' | grep -qE "[:.]$PORT\$"; then
    log "HATA: $PORT kullanımda; PORT=<başka port> ile tekrar deneyin"; exit 2
  fi
  FPM=$(ls /usr/sbin/php-fpm* 2>/dev/null | head -1)
  { [ -n "$FPM" ] && command -v nginx >/dev/null; } || { log "HATA: nginx veya php-fpm bulunamadı"; exit 2; }
  if id -u www-data >/dev/null 2>&1; then WEBU=www-data; else WEBU=nobody; fi
  WG=$(id -gn "$WEBU")
  mkdir -p "$W/web/kok" "$W/web/tmp"
  chmod 755 "$W" "$W/web" "$W/web/kok"
  echo ok > "$W/web/kok/static.txt"
  cat > "$W/web/kok/bench.php" <<'EOF'
<?php
$h = 'vpsbench';
for ($i = 0; $i < 20000; $i++) { $h = hash('sha256', $h, true); }
header('Content-Type: application/json');
echo json_encode(['hash' => bin2hex($h)]);
EOF
  cat > "$W/web/nginx.conf" <<'EOF'
user @WEBU@ @WG@;
worker_processes auto;
pid @W@/web/nginx.pid;
error_log @W@/web/nginx-hata.log;
events { worker_connections 4096; }
http {
    access_log off;
    client_body_temp_path @W@/web/tmp/body;
    fastcgi_temp_path @W@/web/tmp/fastcgi;
    proxy_temp_path @W@/web/tmp/proxy;
    uwsgi_temp_path @W@/web/tmp/uwsgi;
    scgi_temp_path @W@/web/tmp/scgi;
    server {
        listen 127.0.0.1:@PORT@ backlog=4096;
        root @W@/web/kok;
        location ~ \.php$ {
            include /etc/nginx/fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_pass unix:@W@/web/php.sock;
        }
    }
}
EOF
  cat > "$W/web/php-fpm.conf" <<'EOF'
[global]
pid = @W@/web/php-fpm.pid
error_log = @W@/web/php-fpm-hata.log
daemonize = yes

[olcum]
user = @WEBU@
group = @WG@
listen = @W@/web/php.sock
listen.mode = 0666
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
EOF
  sed -i "s|@W@|$W|g; s|@PORT@|$PORT|g; s|@WEBU@|$WEBU|g; s|@WG@|$WG|g" "$W/web/nginx.conf" "$W/web/php-fpm.conf"
  "$FPM" -y "$W/web/php-fpm.conf" || { log "HATA: php-fpm başlamadı"; exit 2; }
  nginx -t -q -c "$W/web/nginx.conf" && nginx -c "$W/web/nginx.conf" || { log "HATA: nginx başlamadı"; exit 2; }
  sleep 1
  if curl -sf "$URL/bench.php" >/dev/null; then
    log "Test sitesi hazır: $URL (yalnız bu makineden erişilir)"
  else
    log "HATA: test sitesi yanıt vermiyor (bkz. $W/web/*-hata.log)"; exit 2
  fi
  ;;

yanit)
  log "Yanıt süresi: tek kullanıcı, bench.php, ${C1} sn"
  ab -t "$C1" -n 1000000 -s 30 -c 1 "$URL/bench.php" > "$OUT/ab_c1.txt" 2>&1
  log "Yanıt süresi: tek kullanıcı, static.txt, ${C1} sn"
  ab -t "$C1" -n 1000000 -s 30 -c 1 "$URL/static.txt" > "$OUT/ab_static_c1.txt" 2>&1
  grep -E "^ +(50|99)%" "$OUT/ab_c1.txt"
  ;;

yuk)
  son=0
  for c in 10 25 50 100 250; do
    log "Yük: $c eş zamanlı kullanıcı, ${KADEME} sn"
    ab -t "$KADEME" -n 1000000 -s 30 -c "$c" "$URL/bench.php" > "$OUT/ab_c$c.txt" 2>&1
    if ab_gecti "$OUT/ab_c$c.txt"; then
      son=$c
    else
      log "$c kullanıcıda düştü (hata > %5 veya p95 > 3 sn)"; break
    fi
  done
  echo "$son" > "$OUT/yuk_son_gecen.txt"
  log "Geçilen son kademe: $son"
  ;;

web-durdur)
  [ -f "$W/web/nginx.pid" ] && kill "$(cat "$W/web/nginx.pid")" 2>/dev/null
  [ -f "$W/web/php-fpm.pid" ] && kill "$(cat "$W/web/php-fpm.pid")" 2>/dev/null
  sleep 1
  log "Test sitesi durduruldu"
  ;;

dayaniklilik)
  rm -f "$OUT/dayaniklilik.bitti"
  log "Sürekli yük: ${DAY} sn, $N iş parçacığı; aynı anda steal örneklemesi"
  steal_ornekle "$DAY" "$DAYARA" "$OUT/steal_yuk.csv" &
  SP=$!
  sysbench cpu --cpu-max-prime=20000 --threads="$N" --time="$DAY" --report-interval="$DAYARA" run > "$OUT/dayaniklilik.txt"
  wait "$SP"
  touch "$OUT/dayaniklilik.bitti"
  log "Dayanıklılık testi bitti"
  ;;

*) log "Bilinmeyen adım: $ADIM"; exit 1 ;;
esac
```

## Ek D: `rapor.py`

```python
#!/usr/bin/env python3
"""VPS-OLCUM sonuç özeti ve skor (v2). Kullanım: python3 rapor.py <out-dizini>"""
import csv
import json
import re
import statistics
import sys
from pathlib import Path

REF = {"cpu": 1438.68, "seq": 2600.35, "iops": 103530.99, "yanit": 43.0, "kademe": 100.0}
KADEMELER = (10, 25, 50, 100, 250)
D = Path(sys.argv[1])


def oku(ad):
    p = D / ad
    return p.read_text(errors="replace") if p.exists() else None


def bul(desen, metin):
    m = re.search(desen, metin or "", re.M)
    return float(m.group(1)) if m else None


def fio_json(ad):
    t = oku(ad)
    if not t or "{" not in t:
        return None
    try:
        return json.loads(t[t.index("{"):])["jobs"][0]
    except (ValueError, KeyError, IndexError):
        return None


def fio(ad, yon):
    j = fio_json(ad)
    if not j or yon not in j:
        return None
    j = j[yon]
    yuzde = j.get("clat_ns", {}).get("percentile", {})
    return {"mb_s": j["bw"] / 1024, "iops": j["iops"], "ort_ms": j["lat_ns"]["mean"] / 1e6,
            "p99_ms": yuzde.get("99.000000", 0) / 1e6}


def fsync(ad):
    j = fio_json(ad)
    lat = (j or {}).get("sync", {}).get("lat_ns", {})
    if not lat:
        return None
    return {"ort_ms": lat.get("mean", 0) / 1e6, "p99_ms": lat.get("percentile", {}).get("99.000000", 0) / 1e6}


def steal(ad):
    t = oku(ad)
    v = [float(r["steal_pct"]) for r in csv.DictReader(t.splitlines())] if t else []
    return {"ort": statistics.mean(v), "en_yuksek": max(v)} if v else None


def gap(ad):
    t = oku(ad)
    try:
        return json.loads(t) if t else None
    except ValueError:
        return None


def surekli(ad):
    eps = [float(x) for x in re.findall(r"eps: ([\d.]+)", oku(ad) or "")]
    if len(eps) < 4:
        return None
    k = max(1, len(eps) // 10)
    ilk, son = statistics.mean(eps[:k]), statistics.mean(eps[-k:])
    return {"ilk": ilk, "son": son, "dusus_pct": 100 * (ilk - son) / ilk,
            "en_kotu_pct": 100 * (ilk - min(eps)) / ilk,
            "cov_pct": 100 * statistics.pstdev(eps) / statistics.mean(eps), "ornek": len(eps)}


def ab(ad):
    t = oku(ad)
    if t is None:
        return None
    tam = bul(r"^Complete requests:\s+(\d+)", t) or 0
    hata = (bul(r"^Failed requests:\s+(\d+)", t) or 0) + (bul(r"^Non-2xx responses:\s+(\d+)", t) or 0)
    return {"istek": int(tam), "rps": bul(r"^Requests per second:\s+([\d.]+)", t),
            "p50_ms": bul(r"^\s+50%\s+(\d+)", t), "p95_ms": bul(r"^\s+95%\s+(\d+)", t),
            "p99_ms": bul(r"^\s+99%\s+(\d+)", t), "hata_pct": 100 * hata / tam if tam else 100.0}


def gecti(r):
    return bool(r) and r["istek"] > 0 and r["hata_pct"] <= 5 and r["p95_ms"] is not None and r["p95_ms"] <= 3000


def mem(ad):
    return bul(r"\(([\d.]+) MiB/sec\)", oku(ad))


def f(v, fmt="{:.1f}"):
    return "ölçülemedi" if v is None else fmt.format(v)


bilgi = json.loads(oku("bilgi.json") or "{}")
n = bilgi.get("vcpu") or 1
t1, tn = bul(r"events per second:\s*([\d.]+)", oku("cpu_t1.txt")), bul(r"events per second:\s*([\d.]+)", oku(f"cpu_t{n}.txt"))
olcek = tn / t1 if t1 and tn else None
verim = 100 * tn / (t1 * n) if t1 and tn else None
gt = gap("gap_tek.json")
gh = [g for g in (gap(p.name) for p in sorted(D.glob("gap_hepsi_*.json"))) if g]
disk = {k: fio(f"fio_{k}.json", "read") for k in ("seqread", "randread", "randread_qd1", "ham_qd1")}
yaz = {k: fio(f"fio_{k}.json", "write") for k in ("seqwrite", "randwrite")}
web = {c: ab(f"ab_c{c}.txt") for c in (1,) + KADEMELER}
statik = ab("ab_static_c1.txt")
son_gecen = 0
for c in KADEMELER:
    if not gecti(web[c]):
        break
    son_gecen = c
su = surekli("dayaniklilik.txt")
ag = (oku("ag.txt") or "").split()
ag_mbit = float(ag[0]) * 8 / 1e6 if ag else None


def p(x):
    return None if x is None else round(min(100.0, max(0.0, x)), 1)


puan = {
    "İşlemci gücü": p(100 * tn / REF["cpu"]) if tn else None,
    "Büyük dosya okuma": p(100 * disk["seqread"]["mb_s"] / REF["seq"]) if disk["seqread"] else None,
    "Küçük parça okuma": p(100 * disk["randread"]["iops"] / REF["iops"]) if disk["randread"] else None,
    "30 dk dayanıklılık": p(100 - max(0.0, su["dusus_pct"])) if su else None,
    "Web yanıt (10 kullanıcı)": p(100 * REF["yanit"] / max(web[10]["p50_ms"], 1))
    if web[10] and web[10]["p50_ms"] is not None else None,
    "Web yük": p(100 * son_gecen / REF["kademe"]) if web[10] else None,
}
bilinen = [v for v in puan.values() if v is not None]
alt, ust = sum(bilinen) / 6, (sum(bilinen) + 100 * (6 - len(bilinen))) / 6

o = ["## Sonuç tablosu", "", "| Ölçüm | Değer |", "|---|---|"]
o += [f"| İşlemci | {bilgi.get('model_adi', '?')} |",
      f"| Mimari, çıkış yılı | {bilgi.get('mimari', '?')}, {bilgi.get('cikis_yili') or '?'} |",
      f"| Sanallaştırma | {bilgi.get('sanallastirma', '?')} |",
      f"| vCPU / RAM (MemTotal) | {n} / {bilgi.get('ram_mb', '?')} MB |",
      f"| Sistem | {bilgi.get('sistem', '?')} |",
      f"| sysbench 1 iş parçacığı | {f(t1)} olay/sn |",
      f"| sysbench {n} iş parçacığı | {f(tn)} olay/sn |",
      f"| Ölçekleme / verim | {f(olcek, '{:.2f}')}x / %{f(verim)} |"]
for ad_, k in (("Steal boşta (ort / en yüksek)", "steal_bosta.csv"), ("Steal yük altında (ort / en yüksek)", "steal_yuk.csv")):
    s = steal(k)
    o.append(f"| {ad_} | " + (f"%{s['ort']:.2f} / %{s['en_yuksek']:.2f}" if s else "ölçülemedi") + " |")
o.append(f"| Gizli duraklama, tek vCPU | " + (f"%{gt.get('lost_pct')} (döngü/µs {gt.get('loops_per_running_us')}, en uzun {gt.get('gap_max_ms')} ms)" if gt else "ölçülemedi") + " |")
if gh:
    o.append(f"| Gizli duraklama, tüm vCPU'lar | %{statistics.mean(g.get('lost_pct', 0) for g in gh):.1f} ortalama "
             f"(döngü/µs {statistics.mean(g.get('loops_per_running_us', 0) for g in gh):.2f}, ortanca boşluk "
             f"{statistics.mean(g.get('gap_median_ms', 0) for g in gh):.2f} ms) |")
for op in ("read", "write"):
    a, b = mem(f"mem_{op}_t1.txt"), mem(f"mem_{op}_t{n}.txt")
    if a or b:
        o.append(f"| RAM {'okuma' if op == 'read' else 'yazma'} (1 / {n} iş parçacığı) | {f(a, '{:,.0f}')} / {f(b, '{:,.0f}')} MiB/s |")
adlar = {"seqread": "Sıralı okuma 1M QD16", "randread": "Rastgele okuma 4K QD32", "randread_qd1": "Rastgele okuma 4K QD1",
         "ham_qd1": "Ham disk okuma 4K QD1", "seqwrite": "Sıralı yazma 1M QD16", "randwrite": "Rastgele yazma 4K QD32"}
for k, r in list(disk.items()) + list(yaz.items()):
    if r:
        o.append(f"| {adlar[k]} | {r['mb_s']:,.0f} MB/s · {r['iops']:,.0f} IOPS · ort {r['ort_ms']:.2f} ms · p99 {r['p99_ms']:.2f} ms |")
fs = fsync("fio_fsync.json")
o.append("| fsync (ort / p99) | " + (f"{fs['ort_ms']:.2f} / {fs['p99_ms']:.2f} ms" if fs else "ölçülemedi") + " |")
o.append(f"| İnternet indirme | {f(ag_mbit, '{:.0f}')} Mbit/s |")
for ad_, r in (("Yanıt, tek kullanıcı, bench.php (p50 / p99)", web[1]), ("Yanıt, tek kullanıcı, static.txt (p50 / p99)", statik)):
    o.append(f"| {ad_} | " + (f"{f(r['p50_ms'], '{:.0f}')} / {f(r['p99_ms'], '{:.0f}')} ms" if r else "ölçülemedi") + " |")
if su:
    o.append(f"| Dayanıklılık | {su['ilk']:.1f} → {su['son']:.1f} olay/sn, düşüş %{su['dusus_pct']:.1f}, "
             f"en kötü anlık %{su['en_kotu_pct']:.1f}, dalgalanma %{su['cov_pct']:.1f} ({su['ornek']} örnek) |")
else:
    o.append("| Dayanıklılık | ölçülemedi |")

o += ["", "## Yük testi", "", "| Kullanıcı | İstek/sn | p50 | p95 | Hata | Sonuç |", "|---|---|---|---|---|---|"]
for c in KADEMELER:
    r = web[c]
    if r:
        o.append(f"| {c} | {f(r['rps'], '{:.0f}')} | {f(r['p50_ms'], '{:.0f}')} ms | {f(r['p95_ms'], '{:.0f}')} ms | "
                 f"%{r['hata_pct']:.1f} | {'geçti' if gecti(r) else 'düştü'} |")
o += ["", f"Geçilen son kademe: **{son_gecen}**", ""]

o += ["## Skor (v2)", "", "| Test | Puan |", "|---|---|"]
o += [f"| {k} | {f(v)} |" for k, v in puan.items()]
o.append("| **Skor** | **" + (f"{alt:.1f}" if alt == ust else f"{alt:.1f} – {ust:.1f} (eksik test 0 / 100 alırsa)") + "** |")
print("\n".join(o))

(D / "ozet.json").write_text(json.dumps({
    "bilgi": bilgi, "sysbench_1": t1, "sysbench_n": tn, "olcekleme": olcek, "verim_pct": verim,
    "steal_bosta": steal("steal_bosta.csv"), "steal_yuk": steal("steal_yuk.csv"),
    "gizli_duraklama_tek": gt, "gizli_duraklama_hepsi": gh, "disk_okuma": disk, "disk_yazma": yaz,
    "fsync": fs, "indirme_mbit": ag_mbit, "web": {str(k): v for k, v in web.items()}, "web_statik": statik,
    "yuk_son_gecen": son_gecen, "dayaniklilik": su, "puan": puan, "skor_alt": alt, "skor_ust": ust,
}, ensure_ascii=False, indent=2))
```

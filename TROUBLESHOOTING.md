# Sorun Giderme (Troubleshooting)

Bu döküman, gerçek sunucu hardening süreçlerinde karşılaşılan tuzakları ve çözümlerini içerir.

---

## T1: HWE Kernel Sonrası Sysctl Parametreleri Resetleniyor

**Belirti:** Reboot sonrası `icmp_echo_ignore_all=0` ve/veya `suid_dumpable=2` oluyor. Sunucu ping'e tekrar cevap veriyor.

**Neden:** HWE kernel geçişinde yeni kernel modülleri `systemd-sysctl.service`'den sonra yükleniyor ve parametreleri varsayılana döndürüyor. Özellikle filesystem modülleri `suid_dumpable`'ı override ediyor.

**Yanlış Çözüm:** `rc.local` kullanmak — Ubuntu 22.04+'da `rc-local.service` varsayılan olarak inactive/dead. Güvenilir çalışmaz.

**Doğru Çözüm:**
```bash
# Systemd unit oluştur
cat > /etc/systemd/system/sysctl-hardening.service << 'EOF'
[Unit]
Description=Apply security sysctl settings after kernel modules
After=systemd-modules-load.service systemd-sysctl.service
DefaultDependencies=no

[Service]
Type=oneshot
ExecStart=/sbin/sysctl -p /etc/sysctl.d/99-hardening.conf
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable sysctl-hardening.service
```

**Doğrulama:** İki ayrı reboot yapıp her ikisinde de parametreleri kontrol et. Tek reboot yetmez.

---

## T2: AIDE Kurulumu Postfix Dependency Hatası

**Belirti:** `apt install aide` komutu Postfix kurmaya çalışıyor ve debconf'ta takılıyor.

**Neden:** AIDE → aide-common → mailx → default-mta (Postfix) dependency zinciri. Postfix hardening'de kaldırılmışsa çakışma olur.

**Çözüm:**
```bash
# Önce s-nail kur (Postfix olmadan mailx alternatifi)
apt-get install -y s-nail

# Postfix sorularını preseed et
echo "postfix postfix/main_mailer_type select No configuration" | debconf-set-selections
echo "postfix postfix/mailname string localhost" | debconf-set-selections

# Sonra AIDE kur
DEBIAN_FRONTEND=noninteractive apt-get install -y aide aide-common
```

---

## T3: aideinit İnteraktif Prompt'ta Takılma

**Belirti:** `aideinit` komutu `[Yn]?` sorarak bekliyor, headless ortamda session kilitlenir.

**Neden:** `aideinit` kendi interaktif promptunu kullanır. `DEBIAN_FRONTEND=noninteractive` bunu bastırmaz (debconf değil).

**Çözüm:**
```bash
yes | aideinit
```

---

## T4: AIDE Baseline Uzun Süre Oluşturma

**Belirti:** `aideinit` CPU %99 kullanarak 5-15 dakika çalışıyor.

**Neden:** Normal davranış. AIDE, `/etc`, `/bin`, `/sbin`, `/usr` altındaki tüm dosyaların SHA-256/SHA-512 hash'ini hesaplıyor.

**Çözüm:** Beklenmesi gereken bir süreçtir. Arka planda çalıştır:
```bash
nohup bash -c 'yes | aideinit && cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db' &
```

---

## T5: debconf İnteraktif Sorular Session'ı Kilitliyor

**Belirti:** `apt install` komutu interaktif sorular soruyor, SSH session'ı kilitlenir.

**Neden:** Bazı paketler (Postfix, tzdata, needrestart) kurulumda interaktif yapılandırma sorar.

**Çözüm:** Tüm apt/dpkg komutlarında environment variable kullan:
```bash
export DEBIAN_FRONTEND=noninteractive

# Veya komut bazında:
DEBIAN_FRONTEND=noninteractive apt-get install -y PAKET_ADI
```

**Ek önlem — needrestart otomatik restart:**
```bash
# needrestart interaktif sormadan otomatik restart yapsın
echo '$nrconf{restart} = "a";' > /etc/needrestart/conf.d/autorestart.conf
```

---

## T6: Postfix Purge Sonrası Failed Service Kalıntıları

**Belirti:** `systemctl --failed` çıktısında Postfix ile ilgili servisler görünüyor.

**Neden:** `dpkg --purge` servis dosyalarını tamamen temizleyemiyor.

**Çözüm:**
```bash
# Failed state'leri temizle
systemctl reset-failed

# Kalıntı servis dosyalarını kontrol et
find /etc/systemd -name "*postfix*" -delete 2>/dev/null
systemctl daemon-reload
```

---

## T7: packagekit Mask'ı needrestart Uyarısı

**Belirti:** packagekit mask'landığında apt kurulumlarında needrestart uyarısı veriyor.

**Neden:** needrestart, packagekit üzerinden servis durumlarını kontrol etmeye çalışıyor.

**Çözüm:** Zararsızdır — uyarıyı görmezden gelebilirsin. Veya:
```bash
echo '$nrconf{restart} = "a";' > /etc/needrestart/conf.d/autorestart.conf
```

---

## T8: dpkg Lock Hatası (Eşzamanlı apt İşlemleri)

**Belirti:** `E: Could not get lock /var/lib/dpkg/lock-frontend`

**Neden:** Başka bir apt/dpkg işlemi çalışıyor (arka plan kurulum veya unattended-upgrades).

**Çözüm:**
```bash
# Çalışan apt işlemlerini kontrol et
ps aux | grep -E "[a]pt|[d]pkg"

# Bekle veya kill et (dikkatli ol):
killall -9 apt-get dpkg 2>/dev/null
rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
dpkg --configure -a
```

---

## T9: SSH Bağlantısı Kesildikten Sonra Sunucuya Erişememe

**Belirti:** Firewall veya SSH config değişikliğinden sonra bağlantı kesildi, yeniden bağlanamıyorsun.

**Önleme:**
1. Config değişikliğinden önce mevcut oturumu KAPATMA
2. Yeni terminal/oturumdan test et
3. Firewall değişikliğinden önce güvenlik çıkışını doğrula

**Kurtarma:**
1. **Tailscale aktifse:** `ssh deployer@TAILSCALE_IP`
2. **Cloudflare Tunnel aktifse:** `ssh sunucu-cf`
3. **Hiçbiri yoksa:** Provider konsol erişimi (recovery password ile)

---

## T10: UFW Kuralları Reboot Sonrası Kaybolma

**Belirti:** Reboot sonrası UFW kuralları sıfırlanmış.

**Neden:** Nadir durum — genellikle UFW kuralları kalıcıdır. Eğer `ufw disable` yapıldıysa kurallar silinir.

**Çözüm:** UFW'yi devre dışı bırakma, sadece `ufw reload` kullan. Kuralları yedekle:
```bash
# Kuralları yedekle
cp /etc/ufw/user.rules /etc/ufw/user.rules.backup
cp /etc/ufw/user6.rules /etc/ufw/user6.rules.backup
```

---

## T11: Cloudflare IP'leri Güncellenmemiş

**Belirti:** Web sitesi aralıklı 502/503 hatası veriyor. Cloudflare yeni IP bloğu eklemiş ama sunucu firewall'unda yok.

**Neden:** Cloudflare IP sync cron'u çalışmamış veya kurulmamış.

**Çözüm:**
```bash
# Cron job'ın varlığını kontrol et
cat /etc/cron.d/cloudflare-ip-sync

# Manuel çalıştır
/usr/local/bin/update-cloudflare-ips.sh

# Cron log kontrolü
grep "cf-ip-sync\|cloudflare" /var/log/syslog | tail -5
```

---

## T12: chrony "Not synchronised" Hatası

**Belirti:** `chronyc tracking` çıktısında "Not synchronised" veya büyük sapma görünüyor.

**Neden:** NTP portları (123/udp) firewall tarafından engellenmiş olabilir veya DNS çözümleme sorunu.

**Çözüm:**
```bash
# chrony durumu
chronyc sources -v

# NTP portunu outgoing olarak açık olduğunu doğrula (UFW default allow outgoing ise sorun yok)
# Eğer outgoing da kısıtlıysa:
ufw allow out 123/udp

# chrony'yi yeniden başlat
systemctl restart chrony
```

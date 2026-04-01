# Faz 4: İzleme ve Denetim Araçları

## Amaç
Çok katmanlı izleme mimarisi kurmak: sistem çağrı loglaması, dosya bütünlüğü izleme, brute-force koruması, zaman senkronizasyonu, log analizi.

## Araç Haritası

| Araç | Görev | Katman |
|------|-------|--------|
| auditd | Sistem çağrı loglaması (kim ne yaptı) | Forensics |
| AIDE | Dosya bütünlüğü izleme (dosya değişti mi) | Tamper detection |
| Fail2Ban | Brute-force algılama ve engelleme | Active defense |
| rkhunter | Rootkit taraması | Malware detection |
| chrony | Zaman senkronizasyonu | Log korelasyonu |
| logwatch | Günlük log özeti | Visibility |
| debsums | Paket bütünlüğü kontrolü | Integrity |

## Adımlar

### 4.1 Tüm Araçları Kur

```bash
export DEBIAN_FRONTEND=noninteractive

# Postfix dependency tuzağını önle — AIDE mailx istiyor, mailx postfix istiyor
# Önce s-nail kur (postfix olmadan mailx alternatifi)
apt-get install -y s-nail

# Postfix sorularını preseed et (dependency olarak kurulursa diye)
echo "postfix postfix/main_mailer_type select No configuration" | debconf-set-selections
echo "postfix postfix/mailname string localhost" | debconf-set-selections

# Tüm araçları kur
apt-get install -y auditd aide aide-common fail2ban rkhunter chrony logwatch debsums lynis

# Postfix kurulduysa devre dışı bırak ve mask'la
systemctl stop postfix 2>/dev/null
systemctl disable postfix 2>/dev/null
systemctl mask postfix 2>/dev/null
```

> **⚠️ AIDE Postfix Dependency Tuzağı:** AIDE → aide-common → mailx → default-mta (Postfix) dependency zinciri var. Postfix'i hardening'de kaldırmışsan, AIDE kurulumu başarısız olabilir. `s-nail` paketi bu sorunu çözer — Postfix olmadan mailx sağlar.

### 4.2 auditd Yapılandırma

```bash
cat > /etc/audit/rules.d/hardening.rules << 'EOF'
# Kimlik dosyaları
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/gshadow -p wa -k identity

# SSH konfigürasyonu
-w /etc/ssh/sshd_config -p wa -k sshconfig
-w /etc/ssh/sshd_config.d/ -p wa -k sshconfig

# Cron
-w /etc/crontab -p wa -k cron
-w /etc/cron.d/ -p wa -k cron
-w /var/spool/cron/ -p wa -k cron

# Firewall
-w /etc/ufw/ -p wa -k firewall

# Sudoers
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# Yetki yükseltme
-w /usr/bin/sudo -p x -k sudo_usage
-w /usr/bin/su -p x -k su_usage

# Kuralları kilitle (reboot'a kadar değiştirilemez)
-e 2
EOF

# Kuralları yükle
systemctl restart auditd
systemctl enable auditd

# Doğrula
auditctl -l | wc -l  # 15+ kural olmalı
```

### 4.3 AIDE Yapılandırma

```bash
# AIDE baseline oluştur
# aideinit interaktif [Yn]? sorar — yes ile pipe'la
yes | aideinit

# Baseline DB'yi yerleştir
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

> **⚠️ AIDE Baseline Süresi:** İlk baseline oluşturma CPU yoğun bir işlemdir. 79GB diskli sunucuda 5-15 dakika sürebilir, CPU %99 kullanır. Bu normaldir — tüm kritik dosyaların SHA-256/SHA-512 hash'ini hesaplıyor.

> **⚠️ aideinit Interaktif Prompt:** `aideinit` komutu `[Yn]?` sorar. `DEBIAN_FRONTEND=noninteractive` bunu bastırmaz (debconf değil, kendi promptu). `yes | aideinit` kullan.

```bash
# Günlük kontrol cron'u (aide-common ile gelir, doğrula)
ls /etc/cron.daily/aide  # Mevcut olmalı

# Manuel kontrol
aide --check  # Değişiklik yoksa "no differences" demeli
```

### 4.4 Fail2Ban Yapılandırma

```bash
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 86400
findtime = 600
maxretry = 3
banaction = ufw

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
EOF

systemctl restart fail2ban
systemctl enable fail2ban

# Güvenli IP whitelist (opsiyonel)
# /etc/fail2ban/jail.local [DEFAULT] altına ekle:
# ignoreip = 127.0.0.1/8 GÜVENLI_IP TAILSCALE_IP
```

### 4.5 chrony Yapılandırma

```bash
systemctl enable chrony
systemctl start chrony

# Doğrula
chronyc tracking | grep -E "Leap status|System time"
# Leap status: Normal olmalı
# System time: birkaç mikrosaniye sapma olmalı
```

### 4.6 rkhunter Yapılandırma

```bash
# Veritabanını güncelle
rkhunter --update
rkhunter --propupd

# İlk tarama
rkhunter --check --sk  # --sk = skip keypress (non-interactive)

# Günlük cron zaten kurulu: /etc/cron.daily/rkhunter
```

### 4.7 logwatch Yapılandırma

```bash
# Günlük özet e-posta yerine dosyaya yaz (Postfix yoksa)
cat > /etc/logwatch/conf/logwatch.conf << 'EOF'
Output = file
Filename = /var/log/logwatch.log
Detail = Med
Range = yesterday
EOF

# Cron zaten kurulu: /etc/cron.daily/00logwatch
```

### 4.8 DNS Koruması (Immutable)

```bash
# Cloudflare DNS
cat > /etc/resolv.conf << 'EOF'
nameserver 1.1.1.1
nameserver 1.0.0.1
EOF

# Immutable yap — DNS hijacking'e karşı koruma
chattr +i /etc/resolv.conf
```

> **Not:** `chattr +i` dosyayı değiştirilemez yapar. Değiştirmek gerekirse önce `chattr -i /etc/resolv.conf` çalıştır.

### 4.9 Doğrulama

```bash
# auditd
auditctl -l | wc -l        # 15+ kural
systemctl is-active auditd  # active

# AIDE
ls -la /var/lib/aide/aide.db  # Mevcut, 20MB+ olmalı
aide --version                 # Versiyon bilgisi

# Fail2Ban
fail2ban-client status        # sshd jail aktif
fail2ban-client status sshd   # Currently banned: 0 (veya daha fazla)

# chrony
chronyc tracking | grep "Leap status"  # Normal

# rkhunter
rkhunter --version  # Versiyon bilgisi

# logwatch
which logwatch  # /usr/sbin/logwatch
```

## LTS Farkları

| Araç | 22.04 | 24.04 |
|------|-------|-------|
| auditd | 3.0.x | 3.1.x (aynı config) |
| AIDE | 0.17 | 0.18 (aynı config) |
| Fail2Ban | 0.11 | 1.0+ (backend değişikliği, config aynı) |
| chrony | 4.2 | 4.5 (aynı config) |

## Sonraki Adım
→ `phases/05-cleanup.md` — Servis temizliği, SUID, AppArmor

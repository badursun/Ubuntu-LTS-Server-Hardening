# Faz 5: Servis Temizliği ve Saldırı Yüzeyi Azaltma

## Amaç
Gereksiz servisleri kaldırmak, tehlikeli SUID bitlerini temizlemek, AppArmor'u etkinleştirmek, swap yapılandırmak.

## Adımlar

### 5.1 Gereksiz Servisleri Kaldır/Devre Dışı Bırak

```bash
export DEBIAN_FRONTEND=noninteractive

# Postfix (SMTP) — port 25 kapatılır
systemctl stop postfix 2>/dev/null
systemctl disable postfix 2>/dev/null
systemctl mask postfix 2>/dev/null
apt-get purge -y postfix 2>/dev/null

# polkitd — lokal yetki yükseltme servisi, sunucuda gereksiz
systemctl stop polkit 2>/dev/null
systemctl disable polkit 2>/dev/null

# packagekitd — GUI paket yöneticisi backend'i
systemctl stop packagekit 2>/dev/null
systemctl disable packagekit 2>/dev/null
systemctl mask packagekit 2>/dev/null
```

> **⚠️ packagekit mask'ı uyarı:** packagekit mask'landığında `needrestart` paketi uyarı verebilir. Bu zararsızdır — needrestart reboot gereksinimini kontrol eden araçtır, packagekit olmadan da çalışır.

> **⚠️ Postfix purge sonrası kalıntılar:** Postfix purge edildikten sonra systemd servis dosyaları kısmen kalabilir ve `systemctl --failed` çıktısında görünebilir. `systemctl reset-failed` ile temizle.

### 5.2 SUID Bitleri Temizle

SUID bitleri, normal kullanıcıların root yetkisiyle program çalıştırmasına izin verir. Sunucuda bunların çoğu gereksizdir.

```bash
# PwnKit CVE-2021-4034 — pkexec SUID'ini kaldır
chmod u-s /usr/bin/pkexec 2>/dev/null

# Diğer gereksiz SUID'ler
chmod u-s /usr/bin/chsh 2>/dev/null
chmod u-s /usr/bin/chfn 2>/dev/null
chmod u-s /usr/bin/newgrp 2>/dev/null
chmod u-s /usr/bin/fusermount3 2>/dev/null

# Mevcut SUID dosyalarını listele (kontrol amaçlı)
find / -perm -4000 -type f 2>/dev/null
```

> **Not:** `sudo`, `su`, `passwd` gibi temel SUID'lere dokunma — bunlar sistem işleyişi için gerekli.

### 5.3 AppArmor Etkinleştir

Ubuntu'da AppArmor varsayılan olarak yüklüdür. Profillerin enforce modunda olduğunu doğrula:

```bash
# Durum kontrolü
aa-status

# Tüm mevcut profilleri enforce moduna al
aa-enforce /etc/apparmor.d/* 2>/dev/null

# Doğrula
aa-status | head -5
# X profiles are loaded.
# X profiles are in enforce mode.
```

### 5.4 Swap Yapılandır

8GB veya altı RAM'li sunucularda swap gerekli — OOM killer kritik servisleri öldürebilir.

```bash
# Mevcut swap kontrolü
free -h | grep Swap
# Swap: 0B ise devam et

# 2GB swap dosyası oluştur
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Kalıcı yap
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Swappiness ayarla (sysctl'de zaten 10 olarak ayarlandı)
sysctl vm.swappiness  # 10 olmalı
```

> **Not:** SSD'li sunucularda swappiness=10 idealdir — RAM tercih eder, swap'ı son çare olarak kullanır. HDD'li sunucularda swappiness=1 daha iyi olabilir.

### 5.5 Sistem Limitleri

```bash
# File descriptor limiti
cat > /etc/security/limits.d/hardening.conf << 'EOF'
* soft nofile 65535
* hard nofile 65535
root soft nofile 65535
root hard nofile 65535
EOF

# Password policy
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
```

### 5.6 Failed Services Temizliği

```bash
# Kalan failed service'leri temizle
systemctl reset-failed

# Doğrula
systemctl --failed --no-pager
# 0 loaded units listed olmalı
```

### 5.7 Doğrulama

```bash
# SUID kontrolü — pkexec SUID'i olmamalı
stat -c '%A' /usr/bin/pkexec  # -rwxr-xr-x (s olmamalı)

# AppArmor
aa-status | grep "profiles are in enforce mode"

# Swap
free -h | grep Swap  # 2.0Gi olmalı
sysctl vm.swappiness  # 10

# Gereksiz servisler
systemctl is-active postfix 2>/dev/null    # inactive veya not found
systemctl is-active packagekit 2>/dev/null  # inactive

# Açık portlar
ss -tlnp | grep -v "127.0.0" | grep -v "::1"
# Sadece SSH portu (2222) görünmeli

# Failed services
systemctl --failed --no-pager  # 0 olmalı
```

## LTS Farkları

| Özellik | 22.04 | 24.04 |
|---------|-------|-------|
| AppArmor | 3.0 | 4.0 (uyumlu) |
| fusermount | fusermount3 | fusermount3 |
| polkit | polkitd | polkitd |
| Swap yöntemi | fallocate | fallocate (aynı) |

## Sonraki Adım
→ `phases/06-verify.md` — Final doğrulama ve güvenlik skoru

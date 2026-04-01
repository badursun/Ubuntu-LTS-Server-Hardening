# Faz 3: Kernel Hardening

## Amaç
Kernel seviyesinde güvenlik parametrelerini sıkılaştırmak ve HWE kernel'e geçiş yapmak.

## HWE Kernel Nedir?

HWE (Hardware Enablement), Ubuntu LTS'in yeni kernel dallarını backport etmesidir. Örneğin Ubuntu 22.04, orijinal 5.15 kernel ile gelir. HWE ile 6.x serisine geçersin. Avantajları:
- Daha geniş güvenlik yaması kapsamı
- Yeni CVE düzeltmeleri
- Güncel donanım desteği

## Adımlar

### 3.1 Mevcut Kernel'i Kontrol Et

```bash
uname -r
# 5.15.0-75-generic → GA kernel, HWE'ye geçiş önerilir
# 6.x.x-xxx-generic → HWE zaten aktif
```

### 3.2 HWE Kernel Kur

```bash
# Ubuntu 22.04:
apt install -y linux-generic-hwe-22.04

# Ubuntu 24.04:
apt install -y linux-generic-hwe-24.04

# Kernel değişimi reboot gerektirir — HENÜZ REBOOT ETME
# Önce sysctl ayarlarını yap (3.3), sonra reboot et
```

### 3.3 Sysctl Güvenlik Parametreleri

```bash
cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
# === ICMP — Görünmezlik ===
net.ipv4.icmp_echo_ignore_all = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# === SYN Flood Koruması ===
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# === IP Spoofing Koruması ===
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# === Redirect/Source Routing Engelleme ===
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# === Martian Packet Logging ===
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# === ASLR ===
kernel.randomize_va_space = 2

# === Core Dump Engelleme ===
fs.suid_dumpable = 0

# === Kernel Bilgi Sızıntısı Engelleme ===
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2

# === Hardlink/Symlink Koruması ===
fs.protected_hardlinks = 1
fs.protected_symlinks = 1

# === IPv6 Tamamen Devre Dışı ===
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# === Performans ===
vm.swappiness = 10
EOF

# Uygula
sysctl -p /etc/sysctl.d/99-hardening.conf
```

### 3.4 Reboot-Proof Systemd Unit

> **⚠️ KRİTİK:** HWE kernel geçişinde bazı kernel modülleri sysctl parametrelerini override eder. Özellikle `icmp_echo_ignore_all` ve `suid_dumpable` varsayılana dönebilir. `rc.local` Ubuntu 22.04+'da güvenilir çalışmaz. Systemd unit kullan.

```bash
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

### 3.5 Reboot ve Doğrulama

```bash
reboot
```

Reboot sonrası:
```bash
# Kernel versiyonu
uname -r  # 6.x.x olmalı

# Kritik sysctl parametreleri
sysctl net.ipv4.icmp_echo_ignore_all     # = 1
sysctl fs.suid_dumpable                  # = 0
sysctl kernel.randomize_va_space         # = 2
sysctl net.ipv4.tcp_syncookies           # = 1
sysctl net.ipv4.conf.all.rp_filter       # = 1
sysctl kernel.dmesg_restrict             # = 1

# Systemd unit durumu
systemctl status sysctl-hardening.service  # active (exited)
```

> **⚠️ Eğer herhangi bir parametre yanlışsa:**
> 1. `sysctl-hardening.service` loglarını kontrol et: `journalctl -u sysctl-hardening.service`
> 2. Parametreyi manuel uygula: `sysctl -w net.ipv4.icmp_echo_ignore_all=1`
> 3. Tekrar reboot edip doğrula — iki temiz boot gerekli

### 3.6 İkinci Reboot Doğrulaması

**Tek reboot yetmez.** Bazı kernel modülleri (özellikle filesystem ile ilgili olanlar) `suid_dumpable`'ı rastgele boot'larda geç aşamada override edebilir. İki temiz boot güven verir:

```bash
reboot
# Tekrar bağlan ve doğrula:
sysctl net.ipv4.icmp_echo_ignore_all fs.suid_dumpable
# Her ikisi de doğruysa → Faz 3 tamamlandı
```

## Parametre Açıklamaları

| Parametre | Değer | Ne Yapar |
|-----------|-------|----------|
| `icmp_echo_ignore_all` | 1 | Ping'e cevap vermez — sunucu görünmez |
| `tcp_syncookies` | 1 | SYN flood saldırılarını engeller |
| `rp_filter` | 1 | Sahte kaynak IP'li paketleri düşürür |
| `randomize_va_space` | 2 | Bellek adreslerini rastgeleleştirir (buffer overflow koruması) |
| `suid_dumpable` | 0 | SUID programların core dump yazmasını engeller |
| `dmesg_restrict` | 1 | Kernel loglarını sadece root görebilir |
| `kptr_restrict` | 2 | Kernel pointer adreslerini gizler |
| `swappiness` | 10 | RAM tercih eder, swap'a az yazar |

## LTS Farkları

| Özellik | 22.04 | 24.04 |
|---------|-------|-------|
| GA Kernel | 5.15 | 6.8 |
| HWE Paket | `linux-generic-hwe-22.04` | `linux-generic-hwe-24.04` |
| sysctl override riski | Yüksek (5.15→6.x atlaması) | Düşük (küçük versiyon farkı) |
| Systemd unit | Gerekli | Gerekli (aynı yöntem) |

## Sonraki Adım
→ `phases/04-monitoring.md` — İzleme ve denetim araçları

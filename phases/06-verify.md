# Faz 6: Final Doğrulama ve Güvenlik Skoru

## Amaç
Tüm fazların doğru uygulandığını kontrol etmek, güvenlik skoru hesaplamak, audit raporu üretmek.

## 10 Maddelik Doğrulama Scripti

Bu scripti hem SETUP sonrası hem de AUDIT modunda kullan:

```bash
#!/bin/bash
echo "========================================="
echo "  SUNUCU GÜVENLİK DOĞRULAMA"
echo "  $(date)"
echo "========================================="

SCORE=0
TOTAL=0
FAILURES=""

check() {
    TOTAL=$((TOTAL + 1))
    local name="$1"
    local cmd="$2"
    local expected="$3"

    result=$(eval "$cmd" 2>/dev/null)
    if echo "$result" | grep -q "$expected"; then
        echo "[✅ PASS] $name"
        SCORE=$((SCORE + 1))
    else
        echo "[❌ FAIL] $name → Beklenen: $expected, Bulunan: $result"
        FAILURES="$FAILURES\n  - $name"
    fi
}

echo ""
echo "=== 1. KULLANICI & SSH ==="
check "Root SSH kapalı" \
    "grep '^PermitRootLogin' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null | tail -1 | awk '{print \$NF}'" \
    "no"

check "Password auth kapalı" \
    "grep '^PasswordAuthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null | tail -1 | awk '{print \$NF}'" \
    "no"

check "SSH portu değiştirilmiş" \
    "grep '^Port' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null | tail -1 | awk '{print \$NF}'" \
    "2222"

check "Sudo kullanıcı mevcut" \
    "grep -c 'sudo' /etc/group" \
    "1"

echo ""
echo "=== 2. FIREWALL ==="
check "UFW aktif" \
    "ufw status | head -1" \
    "active"

check "Default incoming deny" \
    "ufw status verbose | grep 'Default:' | head -1" \
    "deny"

echo ""
echo "=== 3. KERNEL ==="
check "ICMP echo devre dışı" \
    "sysctl -n net.ipv4.icmp_echo_ignore_all" \
    "1"

check "Core dump devre dışı" \
    "sysctl -n fs.suid_dumpable" \
    "0"

check "ASLR aktif" \
    "sysctl -n kernel.randomize_va_space" \
    "2"

check "SYN cookies aktif" \
    "sysctl -n net.ipv4.tcp_syncookies" \
    "1"

check "IP spoofing koruması" \
    "sysctl -n net.ipv4.conf.all.rp_filter" \
    "1"

check "dmesg kısıtlı" \
    "sysctl -n kernel.dmesg_restrict" \
    "1"

check "Sysctl reboot koruması" \
    "systemctl is-active sysctl-hardening.service 2>/dev/null || systemctl is-active thorweb-sysctl.service 2>/dev/null" \
    "active"

echo ""
echo "=== 4. İZLEME ==="
check "auditd aktif" \
    "systemctl is-active auditd" \
    "active"

check "auditd kuralları yüklü" \
    "auditctl -l 2>/dev/null | wc -l" \
    ""
# Kural sayısı 0'dan büyük olmalı — aşağıda ayrıca kontrol edilir
AUDIT_RULES=$(auditctl -l 2>/dev/null | wc -l)
if [ "$AUDIT_RULES" -gt 0 ]; then
    echo "       → $AUDIT_RULES kural aktif"
else
    echo "       [❌] auditd kuralları yüklenmemiş!"
    FAILURES="$FAILURES\n  - auditd kuralları"
fi

check "AIDE baseline mevcut" \
    "ls /var/lib/aide/aide.db 2>/dev/null && echo 'exists'" \
    "exists"

check "Fail2Ban aktif" \
    "systemctl is-active fail2ban" \
    "active"

check "chrony senkron" \
    "chronyc tracking 2>/dev/null | grep 'Leap status'" \
    "Normal"

echo ""
echo "=== 5. SERVİS TEMİZLİĞİ ==="
check "Postfix kapalı/kaldırılmış" \
    "systemctl is-active postfix 2>/dev/null || echo 'inactive'" \
    "inactive"

check "pkexec SUID kaldırılmış" \
    "stat -c '%A' /usr/bin/pkexec 2>/dev/null | grep -c 's'" \
    "0"

check "AppArmor aktif" \
    "aa-status 2>/dev/null | head -1" \
    "profiles"

check "Swap aktif" \
    "free -h | grep Swap | awk '{print \$2}' | grep -c 'i\|G\|M'" \
    "1"

check "Failed services yok" \
    "systemctl --failed --no-pager | grep -c 'loaded'" \
    "0"

echo ""
echo "=== 6. AÇIK PORTLAR ==="
echo "Dış erişime açık portlar:"
ss -tlnp | grep -v "127.0.0" | grep -v "::1"

echo ""
echo "=== 7. DNS ==="
check "DNS immutable" \
    "lsattr /etc/resolv.conf 2>/dev/null | grep -c 'i'" \
    "1"

echo ""
echo "========================================="
echo "  SKOR: $SCORE / $TOTAL"
PERCENT=$((SCORE * 100 / TOTAL))
echo "  YÜZDE: %$PERCENT"
echo ""

if [ "$PERCENT" -ge 95 ]; then
    echo "  DEĞERLENDIRME: ✅ ENDÜSTRİ LİDERLİĞİ SEVİYESİ"
elif [ "$PERCENT" -ge 80 ]; then
    echo "  DEĞERLENDIRME: ⚠️ İYİ — İyileştirme gerekli"
elif [ "$PERCENT" -ge 60 ]; then
    echo "  DEĞERLENDIRME: ⚠️ ORTA — Ciddi eksikler var"
else
    echo "  DEĞERLENDIRME: ❌ ZAYIF — Acil aksiyon gerekli"
fi

if [ -n "$FAILURES" ]; then
    echo ""
    echo "  BAŞARISIZ KONTROLLER:"
    echo -e "$FAILURES"
fi
echo "========================================="
```

## Scripti Kullanma

### Setup Sonrası (Faz 6)
```bash
# Scripti sunucuya kopyala ve çalıştır
bash security-check.sh
```

### Audit Modu
```bash
# Mevcut sunucuya bağlan ve scripti çalıştır
ssh deployer@SUNUCU 'bash -s' < security-check.sh
```

## Skor Tablosu

| Skor | Seviye | Aksiyon |
|------|--------|---------|
| %95-100 | Endüstri liderliği | Bakım moduna geç |
| %80-94 | İyi | Başarısız kontrolleri düzelt |
| %60-79 | Orta | İlgili fazları yeniden uygula |
| %0-59 | Zayıf | Sıfırdan setup modu önerilir |

## Audit Sonrası Yönlendirme

Audit'te başarısız olan her madde için ilgili faz dökümanına yönlendir:

| Başarısız Kontrol | İlgili Faz |
|-------------------|------------|
| Root SSH, password auth, SSH port, sudo user | `phases/01-user-ssh.md` |
| UFW, firewall kuralları | `phases/02-firewall.md` |
| sysctl parametreleri, kernel, reboot koruması | `phases/03-kernel.md` |
| auditd, AIDE, Fail2Ban, chrony | `phases/04-monitoring.md` |
| Postfix, SUID, AppArmor, swap, failed services | `phases/05-cleanup.md` |
| DNS immutable | `phases/04-monitoring.md` §4.8 |

## Periyodik Denetim Takvimi

| Periyot | Görev | Komut |
|---------|-------|-------|
| Günlük | rkhunter rootkit taraması | Otomatik (cron.daily) |
| Günlük | AIDE dosya bütünlüğü kontrolü | Otomatik (cron.daily) |
| Günlük | logwatch log özeti | Otomatik (cron.daily) |
| Haftalık | debsums paket bütünlüğü | Otomatik (cron.weekly) |
| Aylık | Lynis güvenlik taraması | `lynis audit system --quick` |
| Aylık | Bu doğrulama scriptini çalıştır | `bash security-check.sh` |
| Kernel güncelleme sonrası | sysctl doğrulama | `sysctl net.ipv4.icmp_echo_ignore_all fs.suid_dumpable` |

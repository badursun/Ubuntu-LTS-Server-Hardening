# Faz 2: Firewall — Ghost Mode

## Amaç
Sunucuyu dışarıdan tamamen görünmez yapmak. Sadece izin verilen kaynaklardan gelen trafiğe izin vermek.

## Ghost Mode Felsefesi
- **Varsayılan:** Tüm gelen trafik REDDEDİLİR (DROP, REJECT değil — sessiz)
- **İstisnalar:** Sadece açıkça izin verilenler geçer
- **Görünmezlik:** Ping dahil hiçbir şeye cevap vermez — nmap "host down" der

## Adımlar

### 2.1 UFW Temel Yapılandırma

```bash
ufw default deny incoming
ufw default allow outgoing
ufw logging on
```

### 2.2 SSH Erişimi Yapılandırma

Kullanıcının durumuna göre aşağıdaki yöntemlerden birini seç:

#### Yöntem A: Sabit IP ile Erişim (En Basit)

```bash
# Güvenli IP'den SSH'ye izin ver
ufw allow from GÜVENLI_IP to any port 2222 proto tcp comment 'SSH from safe IP'
```

> **Not:** Ev internet bağlantılarında IP değişebilir. Değiştiğinde sunucuya erişim kesilir. Bu riski kabul ediyorsan bu yöntemi kullan, yoksa Yöntem B veya C'ye geç.

#### Yöntem B: Tailscale VPN ile Erişim (Önerilen)

Tailscale, WireGuard tabanlı zero-trust VPN. Sabit IP gerektirmez, NAT arkasında çalışır.

**Sunucuda:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --ssh
```

**UFW'de Tailscale IP'sine tam erişim:**
```bash
# Tailscale IP'sini bul
tailscale ip -4  # Örn: 100.x.x.x

# UFW'de izin ver — bu satırı ASLA silme
ufw allow from TAILSCALE_IP comment 'Tailscale VPN - NEVER REMOVE'
```

> **⚠️ Tailscale kuralı güvenlik çıkışıdır.** Diğer tüm erişim yolları kapansa bile Tailscale üzerinden sunucuya ulaşabilirsin.

**İstemcide (laptop/telefon):**
- https://tailscale.com adresinden uygulamayı indir
- Aynı hesapla giriş yap
- `ssh deployer@TAILSCALE_IP` ile bağlan

#### Yöntem C: Cloudflare Tunnel / Zero Trust (Sabit IP ve VPN Olmadan)

Cloudflare Tunnel, sunucuya dışarıdan gelen bağlantı yerine sunucudan Cloudflare'e çıkan tünel kurar. Hiçbir port açmaya gerek yoktur.

**Kurulum:**
```bash
# cloudflared kur
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | gpg --dearmor -o /usr/share/keyrings/cloudflare-main.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" > /etc/apt/sources.list.d/cloudflared.list
apt update && apt install -y cloudflared

# Tunnel oluştur (Cloudflare hesabı gerekli)
cloudflared tunnel login
cloudflared tunnel create TUNNEL_ADI
```

**SSH erişimi için config:**
```yaml
# /etc/cloudflared/config.yml
tunnel: TUNNEL_ID
credentials-file: /root/.cloudflared/TUNNEL_ID.json
ingress:
  - hostname: ssh.DOMAIN.com
    service: ssh://localhost:2222
  - service: http_status:404
```

```bash
cloudflared service install
systemctl enable cloudflared
```

**İstemcide:**
```bash
# ~/.ssh/config
Host sunucu-cf
    HostName ssh.DOMAIN.com
    ProxyCommand cloudflared access ssh --hostname %h
    User deployer
    IdentityFile ~/.ssh/KEY_DOSYASI
```

> **Avantaj:** Sunucuda SSH portu bile açık olmak zorunda değil. Tüm trafik Cloudflare üzerinden şifreli akar.

#### Yöntem D: Port Knocking (Son Çare)

Diğer yöntemler mümkün değilse, port knocking ile SSH portunu gizle:

```bash
apt install knockd

# /etc/knockd.conf
cat > /etc/knockd.conf << 'EOF'
[options]
    UseSyslog

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 5
    command     = /usr/sbin/ufw allow from %IP% to any port 2222 proto tcp
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 5
    command     = /usr/sbin/ufw delete allow from %IP% to any port 2222 proto tcp
    tcpflags    = syn
EOF

systemctl enable knockd
systemctl start knockd
```

**İstemcide:**
```bash
# Kapıyı çal
knock SUNUCU_IP 7000 8000 9000
# Hemen bağlan
ssh -p 2222 deployer@SUNUCU_IP
# Bitince kapat
knock SUNUCU_IP 9000 8000 7000
```

> **⚠️ Port knocking tek başına güvenlik çıkışı DEĞİLDİR.** Provider konsol erişimini mutlaka yapılandır.

### 2.3 Cloudflare Web Trafiği (80/443)

Eğer sunucu Cloudflare arkasında web trafiği sunuyorsa, sadece Cloudflare IP'lerinden HTTP/HTTPS'e izin ver:

```bash
# Cloudflare IP listesini indir ve UFW'ye ekle
cat > /usr/local/bin/update-cloudflare-ips.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

CF_IPS=$(curl -s https://www.cloudflare.com/ips-v4)

# Mevcut CF kurallarını temizle
ufw status numbered | grep "Cloudflare" | awk -F'[][]' '{print $2}' | sort -rn | while read num; do
    ufw --force delete $num
done

# Yeni kuralları ekle
for ip in $CF_IPS; do
    ufw allow from "$ip" to any port 80,443 proto tcp comment "Cloudflare"
done

ufw reload
echo "[$(date)] Cloudflare IPs updated: $(echo "$CF_IPS" | wc -l) ranges" >> /var/log/cf-ip-sync.log
SCRIPT

chmod +x /usr/local/bin/update-cloudflare-ips.sh

# İlk çalıştırma
/usr/local/bin/update-cloudflare-ips.sh

# Günlük cron
echo "0 3 * * * root /usr/local/bin/update-cloudflare-ips.sh" > /etc/cron.d/cloudflare-ip-sync
```

### 2.4 Cloudflare Origin TLS (Full Strict)

Cloudflare arkasındaki origin sunucu mutlaka TLS ile korunmalı:

```bash
# Origin CA sertifikası oluştur (Cloudflare Dashboard → SSL/TLS → Origin Server)
# Sertifikayı sunucuya koy:
# /etc/ssl/cloudflare/origin.pem (sertifika)
# /etc/ssl/cloudflare/origin-key.pem (private key)

chmod 600 /etc/ssl/cloudflare/origin-key.pem

# Cloudflare Dashboard'da:
# SSL/TLS → Overview → Full (Strict) seç
# SSL/TLS → Edge Certificates → Minimum TLS Version → TLS 1.2
# SSL/TLS → Edge Certificates → Always Use HTTPS → On
```

### 2.5 UFW'yi Etkinleştir

```bash
# Son kontrol — güvenlik çıkışının açık olduğundan emin ol
ufw status verbose

# Etkinleştir
ufw enable
```

### 2.6 Doğrulama

```bash
# UFW durumu
ufw status verbose

# Beklenen çıktı:
# Default: deny (incoming), allow (outgoing), disabled (routed)
# 2222/tcp    ALLOW IN    GÜVENLI_IP (veya Tailscale IP)
# 80,443/tcp  ALLOW IN    Cloudflare IP ranges
# Geri kalan her şey → DROP

# Dışarıdan test (başka bir makineden):
nmap -Pn -p 1-65535 SUNUCU_IP  # Sadece izin verilen portlar görünmeli
ping SUNUCU_IP                  # Cevap gelmemeli (Faz 3'te ayarlanacak)
```

## Provider Konsol Erişimi

**Her durumda provider konsol recovery şifresini kaydet.** SSH, VPN, tunnel hepsi çökerse provider konsolundan erişebilmelisin.

```
Provider Console: https://PROVIDER_URL
Recovery Password: ************ (güvenli yerde sakla)
```

> SSH şifresi devre dışı olsa bile provider konsolu doğrudan sunucu terminaline erişim sağlar. Bu son çare güvenlik çıkışıdır.

## LTS Farkları

| Özellik | 22.04 | 24.04 |
|---------|-------|-------|
| UFW | Tam destek | Tam destek (nftables backend) |
| Tailscale | Tam destek | Tam destek |
| cloudflared | Tam destek | Tam destek |
| knockd | Tam destek | Tam destek |

24.04'te UFW arka planda nftables kullanır ama komutlar aynıdır.

## Sonraki Adım
→ `phases/03-kernel.md` — Kernel hardening ve sysctl yapılandırması

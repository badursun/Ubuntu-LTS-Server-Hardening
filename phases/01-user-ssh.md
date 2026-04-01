# Faz 1: Kullanıcı Oluşturma ve SSH Hardening

## Amaç
Root direkt erişimini kapatıp, ayrı sudo kullanıcı üzerinden key-only SSH erişimi sağlamak.

## Neden Önemli
- Root ile direkt SSH, audit trail oluşturmaz — kim ne yaptı belli olmaz
- `root` brute-force saldırılarında sabit hedef — kullanıcı adı tahmin edilmez
- Sudo kullanıcı ile her yetki yükseltme loglanır

## Adımlar

### 1.1 Sudo Kullanıcı Oluştur

```bash
# Kullanıcı adını belirle (deployer, admin, ops vs.)
adduser deployer
usermod -aG sudo deployer
```

### 1.2 SSH Key'i Yeni Kullanıcıya Kopyala

```bash
mkdir -p /home/deployer/.ssh
cp ~/.ssh/authorized_keys /home/deployer/.ssh/authorized_keys
chown -R deployer:deployer /home/deployer/.ssh
chmod 700 /home/deployer/.ssh
chmod 600 /home/deployer/.ssh/authorized_keys
```

### 1.3 Sudo NOPASSWD (Opsiyonel)

Tek kullanıcılı sunucularda kolaylık için:
```bash
echo "deployer ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/deployer
chmod 440 /etc/sudoers.d/deployer
```

> **Not:** Çok kullanıcılı ortamlarda NOPASSWD kullanma — sudo şifre sorması ek güvenlik katmanıdır.

### 1.4 SSH Hardening

`/etc/ssh/sshd_config` dosyasını düzenle:

```bash
cat > /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# === Port & Protocol ===
Port 2222
AddressFamily inet
# IPv6 devre dışıysa sadece inet kullan

# === Authentication ===
PermitRootLogin no
AllowUsers deployer
MaxAuthTries 3
LoginGraceTime 30
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

# === Forwarding (tümü kapalı) ===
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
GatewayPorts no
PermitTunnel no

# === Hardening ===
Compression no
UseDNS no
DebianBanner no
MaxStartups 3:50:10
ClientAliveInterval 300
ClientAliveCountMax 2

# === Banner ===
Banner /etc/ssh/banner
EOF
```

### 1.5 SSH Banner Oluştur

```bash
cat > /etc/ssh/banner << 'EOF'
*********************************************************************
*  Unauthorized access to this system is prohibited.                *
*  All connections are monitored and recorded.                      *
*  Disconnect IMMEDIATELY if you are not an authorized user.        *
*********************************************************************
EOF
```

### 1.6 OS Banner'ları Temizle

```bash
echo "" > /etc/issue
echo "" > /etc/issue.net
echo "" > /etc/motd
```

### 1.7 SSH Port'unu Firewall'a Ekle (Geçici)

SSH config'i restart etmeden önce yeni portu aç:
```bash
ufw allow 2222/tcp
```

### 1.8 SSH'yi Restart Et ve TEST ET

```bash
systemctl restart sshd
```

> **⚠️ KRİTİK: Mevcut SSH oturumunu KAPATMA!**
> Yeni bir terminal/oturumdan test et:
> ```bash
> ssh -p 2222 deployer@SUNUCU_IP -i ~/.ssh/KEY_DOSYASI
> ```
> Bağlantı başarılıysa eski oturumu kapatabilirsin.
> Başarısızsa mevcut oturumdan düzelt.

### 1.9 Doğrulama

```bash
# Root girişi reddedilmeli
ssh -p 2222 root@SUNUCU_IP  # → Permission denied

# Şifre ile giriş reddedilmeli
ssh -p 2222 -o PubkeyAuthentication=no deployer@SUNUCU_IP  # → Permission denied

# Key ile giriş çalışmalı
ssh -p 2222 deployer@SUNUCU_IP -i ~/.ssh/KEY_DOSYASI  # → Başarılı
```

## LTS Farkları

| Ayar | 22.04 | 24.04 |
|------|-------|-------|
| Config dizini | `/etc/ssh/sshd_config.d/` | Aynı |
| ChallengeResponse | `ChallengeResponseAuthentication` | `KbdInteractiveAuthentication` |
| Varsayılan port | 22 | 22 |

24.04'te `ChallengeResponseAuthentication` yerine `KbdInteractiveAuthentication no` kullan.

## Sonraki Adım
→ `phases/02-firewall.md` — Firewall (Ghost Mode) yapılandırması

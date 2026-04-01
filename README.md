# 🛡️ Server Hardening Skill for Claude Code

Ubuntu LTS sunucuları endüstri liderliği seviyesinde güvenliğe kavuşturan Claude Code skill'i.

Gerçek dünya deneyiminden doğmuştur — teorik değil, savaş alanında test edilmiştir.

---

## Ne Yapar?

Sıfır km bir Ubuntu LTS sunucuyu **~50 dakikada** "Ghost Mode" seviyesine getirir:

| Katman | Koruma |
|--------|--------|
| 👻 **Görünmezlik** | Ping, port scan'e cevap vermez — nmap "host down" der |
| 🔒 **Erişim** | Key-only SSH, custom port, ayrı sudo kullanıcı |
| 🧱 **Firewall** | UFW deny-all + Cloudflare-only web trafiği |
| 🧬 **Kernel** | HWE kernel + 17 sysctl parametresi, reboot-proof |
| 👁️ **İzleme** | auditd + AIDE + Fail2Ban + rkhunter + logwatch |
| 🧹 **Temizlik** | SUID temizliği, Postfix kaldırma, AppArmor enforce |

## Kurulum

```bash
# Claude Code skills dizinine kopyala
cp -r server-hardening ~/.claude/skills/user/server-hardening
```

Veya Claude Code projende:

```bash
mkdir -p .claude/skills/user
cp -r server-hardening .claude/skills/user/
```

## Kullanım

Claude Code'da doğal dilde sor:

```
# Sıfırdan kurulum
"Bu sunucunun güvenliğini sağla"
"Yeni sunucumu hardening yap"

# Mevcut sunucu denetimi
"Sunucumun güvenlik durumunu kontrol et"
"Security audit çalıştır"

# Spesifik konular
"SSH'yi sıkılaştır"
"Firewall'u Ghost Mode yap"
"Kernel hardening ayarlarını uygula"
```

## İki Mod

### 🔧 Setup Modu
Sıfır km sunucu için 6 fazlı kurulum:

```
Faz 1 → Kullanıcı & SSH hardening
Faz 2 → Firewall (Ghost Mode) + erişim yöntemleri  
Faz 3 → HWE kernel + sysctl hardening
Faz 4 → Monitoring (auditd, AIDE, Fail2Ban, chrony)
Faz 5 → Servis temizliği + saldırı yüzeyi azaltma
Faz 6 → 10 maddelik doğrulama + güvenlik skoru
```

### 🔍 Audit Modu
Mevcut sunucu denetimi — kontrol scripti çalıştırır, eksikleri tespit eder, ilgili faza yönlendirir.

## Erişim Yöntemi Desteği

Herkesin sabit IP'si yok. Skill 4 farklı erişim yöntemini destekler:

| Yöntem | Ne Zaman |
|--------|----------|
| **Sabit IP whitelist** | Sabit IP'n varsa — en basit |
| **Tailscale VPN** | Sabit IP yok, VPN kurabilirsin — önerilen |
| **Cloudflare Tunnel** | VPN de kuramıyorsan — zero-trust |
| **Port knocking** | Hiçbiri mümkün değilse — son çare |

Her durumda **provider konsol erişimi** yedek güvenlik çıkışı olarak yapılandırılır.

## Cloudflare Entegrasyonu

- Sadece Cloudflare IP'lerinden HTTP/HTTPS trafiğine izin verir
- Cloudflare IP listesi günlük cron ile otomatik senkronize olur
- Origin CA sertifikası + Full (Strict) modu rehberliği
- TLS 1.2 minimum, Always HTTPS

## Bilinen Tuzaklar (Savaş Alanından)

Skill, gerçek hardening süreçlerinde karşılaşılan 12 tuzağı ve çözümlerini içerir:

| Tuzak | Özetle |
|-------|--------|
| HWE kernel sysctl reset | Kernel modülleri sysctl'i override eder — systemd unit gerekli |
| AIDE Postfix dependency | aide-common → mailx → Postfix zinciri — s-nail ile çözülür |
| aideinit interaktif prompt | `DEBIAN_FRONTEND` çalışmaz — `yes \| aideinit` kullan |
| rc.local çalışmıyor | Ubuntu 22.04+'da inactive — systemd unit kullan |
| debconf session kilidi | Headless ortamda apt takılır — preseed + noninteractive |
| SSH kilitlenme | Config değişikliğinde mevcut oturumu kapatma |

Detaylar: [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md)

## Dosya Yapısı

```
server-hardening/
├── SKILL.md                    # Ana giriş noktası + mod seçimi
├── TROUBLESHOOTING.md          # 12 bilinen tuzak ve çözümleri
└── phases/
    ├── 01-user-ssh.md          # Kullanıcı oluşturma, SSH hardening
    ├── 02-firewall.md          # Ghost Mode, Cloudflare, erişim yöntemleri
    ├── 03-kernel.md            # HWE kernel, sysctl, reboot koruması
    ├── 04-monitoring.md        # auditd, AIDE, Fail2Ban, chrony, logwatch
    ├── 05-cleanup.md           # Servis temizliği, SUID, AppArmor, swap
    └── 06-verify.md            # Doğrulama scripti, skor hesaplama
```

## Desteklenen Sistemler

| Ubuntu LTS | Kernel | Durum |
|------------|--------|-------|
| 22.04 (Jammy) | 5.15 → 6.x HWE | ✅ Tam destek |
| 24.04 (Noble) | 6.8 → HWE | ✅ Tam destek |

Her faz dökümanı LTS sürümleri arasındaki farklılıkları belirtir.

## Hedef Güvenlik Skoru

| Kontrol | Sayı |
|---------|------|
| SSH & erişim kontrolleri | 4 |
| Firewall | 2 |
| Kernel & sysctl parametreleri | 7 |
| İzleme araçları | 5 |
| Servis temizliği | 5 |
| DNS | 1 |
| **Toplam** | **24 kontrol** |

%95+ = Endüstri liderliği seviyesi

## Katkıda Bulunma

PR'lar açıktır. Özellikle:
- Yeni LTS sürümleri için uyumluluk testleri
- Ek troubleshooting maddeleri (gerçek deneyimlerden)
- Alternatif erişim yöntemleri
- Farklı provider'lar için notlar

## Lisans

MIT

---

<p align="center">
<i>Badursun tarafından gerçek dünya sunucu operasyonlarından damıtılmıştır.</i>
</p>

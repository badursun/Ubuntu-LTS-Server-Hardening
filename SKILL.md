---
name: server-hardening
description: "Ubuntu LTS sunucu güvenlik hardening ve denetim skill'i. Bu skill'i şu durumlarda kullan: yeni sunucu kurulumu ve güvenlik yapılandırması, mevcut sunucu güvenlik denetimi (audit), SSH hardening, firewall yapılandırması, kernel güvenlik ayarları, sysctl hardening, UFW Ghost Mode, Cloudflare IP whitelist, AIDE/auditd/fail2ban kurulumu, saldırı yüzeyi azaltma, SUID temizliği, HWE kernel geçişi, sunucu güvenlik skoru hesaplama, penetrasyon testi hazırlığı, server security audit raporu. Tetikleyiciler: sunucu güvenliği, server hardening, güvenlik denetimi, security audit, firewall, SSH, kernel hardening, sysctl, brute force, fail2ban, rootkit, AIDE, auditd, ghost mode, invisible server, Cloudflare origin, saldırı yüzeyi, attack surface, SUID, PwnKit, HWE kernel, güvenlik skoru. Bu skill sunucu ile ilgili herhangi bir güvenlik sorusu veya yapılandırma talebi geldiğinde tetiklenmelidir."
---

# Ubuntu LTS Sunucu Güvenlik Hardening

Endüstri liderliği seviyesinde sunucu güvenliği sağlayan kapsamlı hardening ve denetim rehberi.

## Hedef

Sıfır km bir Ubuntu LTS sunucuyu "Ghost Mode" seviyesine getirmek:
- Dışarıdan görünmez (ping, port scan'e cevap vermez)
- İçeri girilmez (key-only SSH, custom port, Fail2Ban)
- Girilse bile her hareket kaydedilir (auditd, AIDE)
- Bir şeye dokunulursa alarm çalar (dosya bütünlüğü izleme)

## Desteklenen Sistemler

| Ubuntu LTS | Kernel | Durum |
|------------|--------|-------|
| 22.04 | 5.15 → 6.x HWE | Tam destek |
| 24.04 | 6.8 → HWE | Tam destek |
| Gelecek LTS | — | Faz dökümanlarındaki komutlar uyarlanmalı |

**LTS Versiyon Tespiti:**
```bash
lsb_release -cs  # jammy=22.04, noble=24.04
```

## Çalışma Modları

Bu skill iki modda çalışır:

### Mod 1: SETUP — Sıfırdan Kurulum

Yeni sunucuyu 6 fazda endüstri seviyesine getirir. Fazlar sırayla uygulanmalıdır.

| Faz | Döküman | İçerik | Süre |
|-----|---------|--------|------|
| 1 | `phases/01-user-ssh.md` | Kullanıcı oluşturma, SSH hardening | 5 dk |
| 2 | `phases/02-firewall.md` | UFW Ghost Mode, Cloudflare, erişim yöntemleri | 10 dk |
| 3 | `phases/03-kernel.md` | HWE kernel, sysctl, reboot koruması | 10 dk |
| 4 | `phases/04-monitoring.md` | auditd, AIDE, Fail2Ban, chrony, logwatch | 15 dk |
| 5 | `phases/05-cleanup.md` | Servis temizliği, SUID, AppArmor, swap | 5 dk |
| 6 | `phases/06-verify.md` | 10 maddelik doğrulama, skor hesaplama | 5 dk |

**Toplam:** ~50 dakika (AIDE baseline süresi hariç)

Her fazın dökümanını `view` tool ile oku ve adım adım uygula.

### Mod 2: AUDIT — Mevcut Sunucu Denetimi

Mevcut sunucunun güvenlik durumunu kontrol eder ve rapor üretir. `phases/06-verify.md` içindeki audit scriptini çalıştır, sonuçları değerlendir, eksikleri tespit et.

Audit modu sonrası eksik bulunan her madde için ilgili faz dökümanına yönlendir.

## Kritik Kurallar

1. **Her komutu çalıştırmadan önce SSH bağlantısının aktif olduğunu doğrula.** Firewall veya SSH config değişikliğinden sonra mevcut oturumu KAPATMA — yeni bir terminal/oturumdan test et.

2. **Güvenlik çıkışı olmadan firewall sıkılaştırma YAPMA.** Mutlaka aşağıdakilerden en az birini yapılandır:
   - Provider konsol erişimi (recovery password)
   - Tailscale / WireGuard VPN
   - Cloudflare Tunnel (Zero Trust)
   - Port knocking

3. **DEBIAN_FRONTEND=noninteractive** — tüm apt/dpkg komutlarında kullan. Headless sunucularda debconf soruları session'ı kilitleyebilir.

4. **Reboot sonrası her zaman doğrulama yap.** Özellikle kernel değişimlerinde sysctl parametreleri resetlenebilir.

5. **Sıralama önemli:** Faz 1 (SSH) ve Faz 2 (Firewall) tamamlanmadan Faz 3'e geçme. Kilitlenme riski var.

## Sorun Giderme

Bilinen tuzaklar ve çözümleri için `TROUBLESHOOTING.md` dosyasını oku.

## Erişim Yöntemi Karar Ağacı

Sunucuya güvenli erişim için kullanıcının durumuna göre yönlendir:

```
Sabit IP'n var mı?
├── Evet → UFW'de IP whitelist + SSH
├── Hayır → Tailscale/WireGuard kullanabilir misin?
│   ├── Evet → Tailscale kurulumu (phases/02-firewall.md §Tailscale)
│   └── Hayır → Cloudflare Tunnel (Zero Trust) kullan (phases/02-firewall.md §CF-Tunnel)
│       └── Bu da mümkün değilse → Port knocking kur (phases/02-firewall.md §Port-Knocking)
└── Provider konsol erişimi her durumda yedek olarak yapılandır
```

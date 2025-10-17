# OpenVPN İç Ağ (Internal Network) Kurulumu


<img width="632" height="418" alt="openvpn" src="https://github.com/user-attachments/assets/5a90475a-46df-43d4-9be3-9906606501c5" />

Bu doküman, bir **OpenVPN sunucu ve istemci yapısı** kurarak **özel bir iç ağ (TUN arayüzü)** oluşturmayı açıklar.
Bu yapı sayesinde **dış (public) IP adresi değişmeden**, istemciler arasında **şifreli ve güvenli bir özel ağ** oluşturulur.

---

## 1. Kurulum

OpenVPN Road Warrior kurulum betiğini indirip çalıştırın:

```bash
wget https://git.io/vpn -O openvpn-install.sh
sudo chmod +x openvpn-install.sh
sudo bash openvpn-install.sh
```

Kurulum sırasında örnek olarak aşağıdaki yanıtları verebilirsiniz:

```
Which protocol should OpenVPN use?
   1) UDP (recommended)
   2) TCP
Protocol [1]: 1

What port should OpenVPN listen to?
Port [1194]:

Select a DNS server for the clients:
   1) Current system resolvers
   2) Google
   3) 1.1.1.1
   4) OpenDNS
   5) Quad9
   6) AdGuard
DNS server [1]: 2

Enter a name for the first client:
Name [client]: iphone
```

Kurulum tamamlandığında, ilk istemci için `.ovpn` yapılandırma dosyası otomatik olarak oluşturulacaktır.

---

## 2. Sunucu Yapılandırması

Sunucu yapılandırma dosyası `/etc/openvpn/server.conf` konumundadır.
Aşağıda örnek bir yapılandırma gösterilmiştir:

```bash
local your_server_ip
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
auth SHA512
cipher AES-256-GCM
tls-crypt tc.key
topology subnet
server 100.100.0.0 255.255.255.0
push "route 100.100.0.0 255.255.255.0"
ifconfig-pool-persist ipp.txt
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"
push "block-outside-dns"
keepalive 10 120
user nobody
group nogroup
persist-key
persist-tun
verb 3
crl-verify crl.pem
explicit-exit-notify
status /etc/openvpn/openvpn-status.log
status-version 2
log-append /var/log/openvpn.log
client-to-client
```

> **Not:**
> `client-to-client` satırı, VPN istemcilerinin birbirleriyle doğrudan iletişim kurmasını sağlar.
> Bu özellik kapatılmak istenirse satır kaldırılabilir.

---

## 3. İstemci Yapılandırması

Oluşturulan `.ovpn` dosyalarını istemcilere kopyalayın:

```bash
sudo mkdir -p /etc/openvpn/client
sudo cp /path/to/client.ovpn /etc/openvpn/client/client.ovpn
```

---

## 4. Systemd Servis Tanımı (İstemci)

İstemci bağlantısını sistem servisi olarak çalıştırmak için aşağıdaki adımları izleyin:

```bash
sudo nano /etc/systemd/system/openvpn-client@client.service
```

İçeriği şu şekilde ekleyin:

```ini
[Unit]
Description=OpenVPN client for %i
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/openvpn --config /etc/openvpn/client/%i.ovpn
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Servisi etkinleştirin ve başlatın:

```bash
sudo systemctl daemon-reload
sudo systemctl enable openvpn-client@client
sudo systemctl start openvpn-client@client
sudo systemctl status openvpn-client@client
```

---

## 5. Ağ Topolojisi

Bu yapılandırma ile tüm istemciler aşağıdaki örnek özel ağ üzerinden iletişim kurar:

```
Sunucu:   100.100.0.1
İstemci-1: 100.100.0.2
İstemci-2: 100.100.0.3
...
```

Dış IP adresi değişmeden, yalnızca VPN tüneli üzerinden geçen trafik etkilenir.

---

## 6. Kullanım Senaryoları

### 6.1 Güvenli SSH Erişimi

Yönetim erişimi için SSH bağlantılarını doğrudan VPN iç IP adresleri üzerinden kurabilirsiniz:

```bash
ssh root@100.100.0.2
```

Bu sayede 22 numaralı portu internete açmanıza gerek kalmaz.

---

### 6.2 Servis Replikasyonu ve Senkronizasyonu

Veritabanı veya önbellek servisleri (MySQL, Redis, MongoDB vb.) arasında güvenli replikasyon kurulabilir:

```
primary-db (100.100.0.2) <--> replica-db (100.100.0.3)
```

Tüm veri trafiği VPN tüneli içinde şifrelenmiş olarak aktarılır.

---

### 6.3 Uygulamalar Arası İç Trafik

Web veya API sunucuları yalnızca VPN iç ağı üzerinden birbirine veri gönderebilir.
Bu yöntem, load balancer veya WAF arkasındaki sistemlerde dahili iletişimi güvenli hale getirir.

---

## 7. Log ve Durum Kontrolü

Log ve bağlantı durumlarını kontrol etmek için:

```bash
sudo tail -f /var/log/openvpn.log
cat /etc/openvpn/openvpn-status.log
```

---

## 8. Route Çakışma Sorunu ve Çözümü

Bazı sistemlerde aynı anda birden fazla VPN bağlantısı (örneğin bir ofis VPN’i ve bir OpenVPN istemcisi) aktif olduğunda,
Windows tarafında **route çakışması (IP yönlendirme çakışması)** yaşanabilir.

Bu durumda tünel ağı (örneğin `100.100.0.0/24`) diğer VPN’in rotalarıyla karışabilir ve istemciler VPN ağına erişemez.

### Sorunun Belirtileri

* OpenVPN bağlantısı aktif görünmesine rağmen `ping 100.100.0.1` cevapsız kalır.
* `Get-NetRoute` çıktısında aynı ağ için birden fazla yönlendirme görünür.

### Çözüm Adımları (Windows)

1. Tünel arayüzlerinin ID’lerini listeleyin:

   ```powershell
   Get-NetIPInterface
   ```

2. VPN arayüzlerinin metrik değerlerini düzenleyin:

   ```powershell
   Set-NetIPInterface -InterfaceIndex 11 -InterfaceMetric 500   # Önceliği düşük VPN
   Set-NetIPInterface -InterfaceIndex 66 -InterfaceMetric 1     # OpenVPN tüneli (öncelikli)
   ```

3. Gerekirse yönlendirmeyi manuel olarak oluşturun:

   ```powershell
   New-NetRoute -DestinationPrefix 100.100.0.0/24 -InterfaceIndex 66 -NextHop 100.100.0.1 -RouteMetric 1 -PolicyStore ActiveStore
   ```

> Eğer “Instance MSFT_NetRoute already exists” hatası alırsanız, bu route zaten mevcut demektir.
> Bu durumda, aşağıdaki komutla silip yeniden oluşturabilirsiniz:
>
> ```powershell
> Remove-NetRoute -DestinationPrefix 100.100.0.0/24 -Confirm:$false
> New-NetRoute -DestinationPrefix 100.100.0.0/24 -InterfaceIndex 66 -NextHop 100.100.0.1 -RouteMetric 1 -PolicyStore ActiveStore
> ```

Bu şekilde, sistem hangi VPN bağlantısının hangi ağı yönlendireceğini açıkça bilir
ve **çakışma, bağlantı kaybı veya yönlendirme hataları ortadan kalkar.**

---

## 9. Güvenlik Önerileri

* Güvenlik duvarında yalnızca `1194/udp` portunu açık bırakın.
* `client-to-client` özelliğini ihtiyacınız yoksa devre dışı bırakın.
* `.ovpn` dosyalarını ve sertifikaları üçüncü kişilerle paylaşmayın.
* Mümkünse IP yerine domain tabanlı bağlantı kullanın.

---

## 10. Özet

Bu yapılandırma ile:

* OpenVPN iç ağı güvenli bir tünel üzerinden çalışır,
* İstemciler birbiriyle doğrudan haberleşebilir,
* Birden fazla VPN kullanımı durumunda route çakışmaları giderilir,
* Sistem yönlendirmeleri manuel olarak kontrol altına alınır.

---

## Okudğunuz için teşekkürler.

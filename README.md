# 🧩 OpenVPN Internal Network Setup (Private Network Layer)

Bu doküman, kendi özel (internal) ağınızı oluşturmak için **OpenVPN Road Warrior Installer** kullanarak bir **OpenVPN sunucu ve istemci** yapısı kurmayı açıklar.
Yapı, **dış (public) IP değişmeden**, istemciler arasında **özel bir sanal ağ (TUN)** oluşturur.

---

## 🚀 1. Kurulum Script’ini İndir ve Çalıştır

```bash
wget https://git.io/vpn -O openvpn-install.sh
sudo chmod +x openvpn-install.sh
sudo bash openvpn-install.sh
```

Kurulum sırasında aşağıdaki örnek yanıtları verebilirsiniz:

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

Kurulum tamamlandığında, ilk istemci için `.ovpn` yapılandırma dosyası oluşturulur.

---

## ⚙️ 2. OpenVPN Server Konfigürasyonu

Sunucu konfigürasyonu `/etc/openvpn/server.conf` dosyasında yer alır:

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
#push "redirect-gateway def1 bypass-dhcp"
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

🟢 **Not:**
Bu yapı istemciler arasında **client-to-client** iletişimini aktif eder, yani VPN üzerinden birbirleriyle doğrudan iletişim kurabilirler.

---

## 📦 3. İstemci (Client) Yapılandırması

Oluşturulan `.ovpn` dosyalarını istemcilere kopyalayın:

```bash
sudo mkdir -p /etc/openvpn/client
sudo cp /path/to/client.ovpn /etc/openvpn/client/client.ovpn
```

---

## 🧠 4. Systemd Üzerinden OpenVPN Client Servisi Oluşturma

Yeni bir systemd servis dosyası oluşturun:

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

Servisi aktif hale getirin:

```bash
sudo systemctl daemon-reload
sudo systemctl enable openvpn-client@client
sudo systemctl start openvpn-client@client
sudo systemctl status openvpn-client@client
```

---

## 🌐 5. Network Topolojisi

Bu yapı ile VPN istemcileri, aşağıdaki gibi **özel bir iç IP aralığı** üzerinden haberleşir:

```
Server: 100.100.0.1
Client-1: 100.100.0.2
Client-2: 100.100.0.3
...
```

🔒 Dış IP adresi değişmez; yalnızca VPN tüneli içindeki trafiği etkiler.
Bu yöntem genellikle **servislerin birbirine güvenli erişimi** veya **özel SSH bağlantıları** için kullanılır.

---

## 🧩 6. Örnek Kullanım Senaryoları

Aşağıdaki örnekler, OpenVPN’in dahili ağ oluşturma gücünü gösterir:

### 🖥️ 1. Güvenli SSH Erişimi

Birden fazla uzak sunucuya erişirken, sadece VPN iç IP’lerini kullanarak SSH bağlantısı kurabilirsiniz:

```bash
ssh root@100.100.0.2
```

Bu sayede dış portları (22) internet’e açmadan yönetim yapabilirsiniz.

---

### ⚙️ 2. Servis Replikasyonu

Veritabanı veya cache servisleri (MySQL, Redis, MongoDB vb.) arasında yalnızca VPN iç IP’leriyle replikasyon yapılabilir:

```
primary-db (100.100.0.2) <--> replica-db (100.100.0.3)
```

Böylece hem trafik şifrelenmiş olur hem de güvenli bir replikasyon ağı kurulur.

---

### 🌐 3. Uygulama Sunucuları Arasında İç Trafik

Web veya API sunucularınız, sadece VPN ağı üzerinden birbirine veri gönderebilir.
Bu yöntem, **load balancer** veya **WAF** arkasında kalan sistemlerde güvenli dahili iletişim sağlar.

---

## 📜 Log ve Durum Kontrolü

```bash
sudo tail -f /var/log/openvpn.log
cat /etc/openvpn/openvpn-status.log
```

---

## 🧱 Güvenlik İpuçları

* `ufw` veya `iptables` üzerinden yalnızca `1194/udp` portunu açın.
* Gerekiyorsa `client-to-client` satırını kaldırarak istemciler arası iletişimi kapatabilirsiniz.
* `.ovpn` dosyalarını paylaşmadan önce **sertifikaları koruyun**.
* Sunucu IP’sini gizli tutmak için domain tabanlı bağlantı tercih edin.

---

Okuduğunuz için teşekkürler.

---


# Jarkom-Modul-3-2025-K27

---

## 👥 Anggota Kelompok
| Nama | NRP |
|------|------|
| Naila Raniyah Hanan | 5027241078|
| Naufan Andi | 50272410 |

---

## 🧠 Deskripsi Umum
  simulasi praktikum jaringan komputer tingkat lanjut yang menggunakan narasi epik      dari The Last Alliance (dari dunia fantasi The Lord of the Rings) sebagai metafora    untuk mengimplementasikan arsitektur layanan web terdistribusi dan layanan            infrastruktur (DHCP, DNS, Routing) dalam lingkungan virtualisasi (Docker/GNS3).

  
---

## 🗺️ Topologi

  <img width="713" height="513" alt="image" src="https://github.com/user-attachments/assets/d05fb03d-c895-458a-af0a-dfcb143117f7" />


---

## ⚙️ Soa2 & Penyelesaian

---
### **2. Aldarion & Durin: Pembagian Tanah Jaringan (DHCP Relay)**
---
#### 🔹 Jawaban & Tata Cara
mendirikan sistem pembagian alamat IP otomatis. Aldarion (DHCP Server) bertindak sebagai juru tulis yang mencatat dan membagi "tanah" (alamat IP) secara adil. Durin (Router/DHCP Relay) bertugas menyampaikan pesan permintaan "tanah" dari klien di seluruh kerajaan kembali ke Aldarion.

1. Konfigurasi Aldarion (DHCP Server)
   ```bash
   # Run on aldarion
    # 1. Pasang perangkat lunak pembagi alamat IP
    apt-get update
    apt-get install isc-dhcp-server -y

    # Konfigurasi Interface yang terhubung ke jaringan internal (Ganti eth0 jika perlu)
    cat > /etc/default/isc-dhcp-server << EOF
    INTERFACESv4="eth0"
    EOF

    # 2. Atur Daftar Pembagian Tanah (dhcpd.conf)
    cat > /etc/dhcp/dhcpd.conf << EOF
    # Opsi global sederhana
    ddns-update-style none;

    # Manusia: Mendapatkan IP di Subnet 1 (10.77.1.x)
    subnet 10.77.1.0 netmask 255.255.255.0 {
        range 10.77.1.6 10.77.1.34;
        range 10.77.1.68 10.77.1.94;
        option routers 10.77.1.1; # Gerbang/Router di Subnet ini
        option broadcast-address 10.77.1.255;
        option domain-name-servers 10.77.3.3; # Alamat DNS Master (Erendis)
    }

    # Peri: Mendapatkan IP di Subnet 2 (10.77.2.x)
    subnet 10.77.2.0 netmask 255.255.255.0 {
       range 10.77.2.35 10.77.2.67;
       range 10.77.2.96 10.77.2.121;
       option routers 10.77.2.1; # Gerbang/Router di Subnet ini
       option broadcast-address 10.77.2.255;
       option domain-name-servers 10.77.3.3; # Alamat DNS Master (Erendis)
    }

    # Subnet 3: Untuk alamat statis dan Fixed Address Khamul
    subnet 10.77.3.0 netmask 255.255.255.0 {
        option routers 10.77.3.1;
        option broadcast-address 10.77.3.255;
        option domain-name-servers 10.77.3.3;
    }
    
    # Subnet 4: Jaringan Aldarion (Server DHCP)
    subnet 10.77.4.0 netmask 255.255.255.0 {
        option routers 10.77.4.1;
        option broadcast-address 10.77.4.255;
        option domain-name-servers 10.77.3.3;
    }
    
    # Khamul yang misterius: Diberi alamat IP tetap
    host Khamul {
        hardware ethernet 02:42:dc:08:82:00; # GANTI dengan MAC Address Khamul
        fixed-address 10.77.3.95;
    }
    EOF
    # 3. Aktifkan Layanan
    service isc-dhcp-server restart
   
   ```
2. Konfigurasi Durin (DHCP Relay)
   ```bash
    # Run on durin
    # 1. Pasang perangkat lunak DHCP Relay
    apt-get update
    apt-get install isc-dhcp-relay -y
    
    # 2. Konfigurasi Relay: Tahu siapa Aldarion (Server DHCP) dan harus dengar di mana (Interfaces)
    cat > /etc/default/isc-dhcp-relay << EOF
    SERVERS="10.77.4.2" # IP Aldarion (DHCP Server)
    INTERFACE="eth1 eth2 eth3" # Interface yang terhubung ke jaringan klien (sesuaikan)
    OPTIONS=""
    EOF
    
    # 3. Aktifkan Durin sebagai Router (meneruskan paket antar subnet)
    cat > /etc/sysctl.conf << EOF
    net.ipv4.ip_forward=1
    EOF
    
    sysctl -p
    # 4. Aktifkan Layanan Relay
    service isc-dhcp-relay restart
       
   ```
   <img width="568" height="338" alt="image" src="https://github.com/user-attachments/assets/8bfbfe53-e63c-4748-861e-cd8b1c51807a" />
   <img width="565" height="312" alt="image" src="https://github.com/user-attachments/assets/244e86d2-12ff-44cf-9ddc-42b71b190834" />
<img width="562" height="339" alt="image" src="https://github.com/user-attachments/assets/276f034d-f67b-4ac4-812b-a7d8b1e4c5f6" />


   ## ⚙️ Soa3 & Penyelesaian
     Untuk mengontrol arus informasi ke dunia luar (Valinor/Internet), sebuah          menara pengawas, Minastir didirikan. Minastir mengatur agar semua node            (kecuali Durin) hanya dapat mengirim pesan ke luar Arda setelah melewati          pemeriksaan di Minastir.

   ---
   ### **3. Minastir: Menara Pengawas Informasi (DNS Forwarder)**
   --- 
    Minastir bertindak sebagai menara pengawas, gerbang menuju dunia luar             (Valinor/Internet). Semua permintaan informasi dari luar domain lokal             (k27.com) harus melewati Minastir.
   1. Konfigurasi Minastir (DNS Forwarder).
      ```
            # Run on minastir
      # 1. Pasang BIND9 (perangkat lunak DNS)
      apt-get update
      apt-get install bind9 -y
      
      # 2. Atur Minastir untuk Meneruskan permintaan ke DNS Publik (8.8.8.8)
      # Minastir akan mencari tahu alamat luar, bukan menyimpannya.
      cat > /etc/bind/named.conf.options << EOF
      options {
          directory "/var/cache/bind";
          recursion yes;
      
          // Meneruskan semua permintaan ke DNS Google (Valinor/Internet)
          forwarders {
              8.8.8.8;
              8.8.4.4;
          };
      
          // Kebijakan: Hanya gunakan forwarder (tidak mencoba cari sendiri)
          forward only; 
      
          allow-query { any; };
          listen-on { any; };
          listen-on-v6 { none; };
      };
      EOF
      
      # 3. Aktifkan Layanan
      service bind9 restart
      
      # 4. Konfigurasi Klien: Arahkan semua Node ke Minastir (10.77.5.2)
      # Ini harus dilakukan di SEMUA NODE (kecuali router Durin)
      # IP Minastir: 10.77.5.2
      echo "nameserver 10.77.5.2" | tee /etc/resolv.conf
      ```
      <img width="558" height="312" alt="image" src="https://github.com/user-attachments/assets/4b693fa4-09af-4efa-9af3-b7f23da37798" />

   

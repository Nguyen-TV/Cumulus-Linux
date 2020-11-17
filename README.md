Hệ thống mạng GalaxyPlay đang dùng BGP EVPN.
Các leaf,spine và edge dùng BGP unnumbered làm underlay và dùng EVPN làm overlay.
Các server được cấu hình bond tới các cặp leaf. Trên các cặp leaf, các port đấu xuống server được cấu hình MLAG bond.

Trên cặp Edge thiết lập neighbor với router của FPT và nhận default route từ FPT.
Đồng thời trên cặp edge có access list để filter các network được quảng bá vào mạng local,
nhằm tránh trường hợp FPT quảng bá full bảng route vào hệ thống mạng GalaxyPlay.

**Cấu hình switch đang sử dụng**

Edge switch: Dell S6010-ON (Cumulus 3.7.12)

Spine switch: HP Altoline 6712-32QSFP+ (Cumulus 3.7.12)

Leaf switch: Dell S4048-ON (Cumulus 3.7.12)

CHÚ Ý: Các file cấu hình này được dùng trên Cumulus 3.7.12! Đối với Cumulus 4 sẽ có chút điều chỉnh file config switchd.conf.


**Topology**

![](Fim+_Topo-eBGP_and_VXLAN.jpg)

Các switch sẽ được cấp IP tự động nếu reset cấu hình bằng lệnh "net del all".

Các IP này được pfsense reserve IP dựa theo địa chỉ MAC của eth0 interface.

Nên sau khi reset cấu hình switch thì switch sẽ nhận lại đúng IP eth0 cũ.

User/pass không thay đổi sau khi reset cấu hình. User/pass chỉ thay đổi sau khi cài lại hoặc upgrade OS.


Trước khi chạy ansible cho switch thì phải cấu hình cơ bản trước. Phải cấu hình hostname đúng cho switch vì ansible chạy dựa trên hostname hiện tại trên switch.

**Đối với Edge switch:**

```
net add dns nameserver ipv4 8.8.8.8
net add time zone Asia/Ho_Chi_Minh
net del time ntp server 0.cumulusnetworks.pool.ntp.org
net del time ntp server 1.cumulusnetworks.pool.ntp.org
net del time ntp server 2.cumulusnetworks.pool.ntp.org
net del time ntp server 3.cumulusnetworks.pool.ntp.org
net add time ntp server 10.10.10.1 iburst
net add hostname edge01/edge02
net commit
```



**Đối với spine và leaf switch:**

```
net add dns nameserver ipv4 8.8.8.8
net add time zone Asia/Ho_Chi_Minh
net del time ntp server 0.cumulusnetworks.pool.ntp.org
net del time ntp server 1.cumulusnetworks.pool.ntp.org
net del time ntp server 2.cumulusnetworks.pool.ntp.org
net del time ntp server 3.cumulusnetworks.pool.ntp.org
net add time ntp server 10.10.10.1 iburst
net add hostname leaf/spine
net commit
```

**Cấu hình bổ sung:**
Đối với Edge thì điều chỉnh switchd.conf
```
vi /etc/cumulus/switchd.conf 
# configure a route instead of a neighbor with the same ip/mask
#route.route_preferred_over_neigh = FALSE
route.route_preferred_over_neigh = TRUE
```
Sau khi chỉnh sửa file switchd.conf thì phải restart service switchd

Mặc định trên Cumulus 4 thì giá trị này là TRUE. Trên Cumulus 3.7 phải điều chỉnh lại.

Tham số này giúp BGP detect neighbor tốt hơn. Khi 1 host down thì sẽ update route nhanh hơn.


**Anycast config on Server**
Đặt IP loopback và cài đặt quagga

```
/etc/quagga/bgpd.conf
router bgp 63732
bgp router-id 103.205.107.15 #Physical IP của server01
network 103.205.107.17/32 #Ip loopback là IP anycast
neighbor 103.205.107.2 remote-as 63732
neighbor 103.205.107.3 remote-as 63732
```

**Anycast on Edge**
```
net add bgp neighbor 103.205.107.15 remote-as 63732
net add bgp neighbor 103.205.107.16 remote-as 63732

net add bgp ipv4 unicast neighbor 103.205.107.15 route-map anycast-network out
net add bgp ipv4 unicast neighbor 103.205.107.16 route-map anycast-network out
```
Trong đó .15 và .16 là IP vật lý của 2 server.

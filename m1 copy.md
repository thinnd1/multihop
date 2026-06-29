Đây là tổng hợp các khái niệm mạng máy tính theo từng lớp và chủ đề chính:Dưới đây là tóm tắt nhanh theo từng nhóm:

**Mô hình OSI (7 tầng)**
Từ dưới lên: Vật lý → Liên kết dữ liệu → Mạng → Vận chuyển → Phiên → Trình bày → Ứng dụng. Mô hình TCP/IP thực tế gộp lại còn 4 tầng: Network Access, Internet, Transport, Application.

**Địa chỉ & định tuyến**
IPv4 (32-bit, ví dụ 192.168.1.1), IPv6 (128-bit). CIDR notation như /24 xác định subnet mask. NAT cho phép nhiều thiết bị dùng chung 1 IP public. Routing quyết định đường đi của gói tin qua các router.

**Giao thức vận chuyển**
TCP đảm bảo tin cậy qua 3-way handshake (SYN → SYN-ACK → ACK), kiểm soát luồng và tắc nghẽn. UDP nhanh hơn nhưng không đảm bảo — dùng cho streaming, DNS, game.

**Tầng ứng dụng**
HTTP/HTTPS (web), DNS (phân giải tên miền), SMTP/POP3/IMAP (email), FTP/SFTP (truyền file), SSH (remote), DHCP (cấp IP tự động).

**Bảo mật mạng**
Firewall lọc traffic theo rule. VPN mã hóa kết nối. TLS/SSL bảo vệ dữ liệu truyền (HTTPS = HTTP + TLS). IDS/IPS phát hiện và ngăn xâm nhập. ACL kiểm soát quyền truy cập.

**Công cụ debug thường dùng**
`ping` kiểm tra kết nối, `traceroute`/`tracert` xem đường đi gói tin, `nslookup`/`dig` tra DNS, `netstat` xem port đang mở, `tcpdump`/Wireshark bắt và phân tích gói tin.

Bạn muốn đào sâu vào chủ đề nào không? Ví dụ TCP handshake, subnetting, hay bảo mật mạng?
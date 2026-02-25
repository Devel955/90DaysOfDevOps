Day 15 – Networking Concepts (My Practice Notes)
Task 1: DNS – How Domain Works
Jab main browser me google.com likhta hoon, kya hota hai?

Sabse pehle browser apne cache me IP check karta hai.
Agar nahi milta, to DNS server se poochta hai.
DNS server google ka IP deta hai.
Phir browser us IP se connect ho jata hai.

DNS Records
A Record – Domain ko IPv4 address deta hai
AAAA Record – Domain ko IPv6 deta hai
CNAME – Ek domain ko dusre domain se jodta hai
MX – Mail server batata hai
NS – Name server batata hai
Command Used
dig google.com
Output (Important Part)
google.com.   222   IN   A   142.250.74.14

IP Address: 142.250.74.14
TTL: 222 seconds

Iska matlab DNS 222 seconds tak is IP ko yaad rakhega.
Task 2: IP Addressing
IPv4 Address Kya Hota Hai?
IPv4 ek unique number hota hai jo har device ko network me identify karta hai.
Example: 192.168.1.10
Public vs Private IP
Public IP: Internet pe directly accessible
Example: 8.8.8.
Private IP: Internal network ke liye hota hai
Example: 172.31.20.14
Private IP Ranges
10.0.0.0 – 10.255.255.255
172.16.0.0 – 172.31.255.255
192.168.0.0 – 192.168.255.255

Command Used
ip addr show
My System Output (Important Part)
inet 172.31.20.14/20
inet 172.17.0.1/16
My Private IPs
Interface	IP Address	Type
ens5	172.31.20.14	Private IP
docker0	172.17.0.1	Private IP
Task 3: CIDR & Subnetting
/24 Ka Matlab
/24 ka matlab hota hai 24 bits network ke liye use ho rahe hain
Aur baaki bits hosts ke liye.

Usable Hosts
CIDR	Total IPs	Usable
/24	256	254
/16	65536	65534
/28	16	14
Subnetting Kyu Karte Hain?
Network ko small parts me divide karne ke liye
Traffic kam karne ke liye
Security improve karne ke liye

CIDR Table
CIDR	Subnet Mask	Total IPs	Usable Hosts
/24	255.255.255.0	256	254
/16	255.255.0.0	65536	65534
/28	255.255.255.240	16	14
Task 4: Ports – Services Ke Darwaze
Port Kya Hota Hai?

Port batata hai ki kaunsa service data receive karega.
Ek hi IP pe multiple services chal sakte hain ports ke wajah se.

Common Ports
Port	Service
22	SSH
80	HTTP
443	HTTPS
53	DNS
3306	MySQL
6379	Redis
27017	MongoDB
Command
ss -tulpn

(Example: SSH aur MySQL listening mode me mile)

Task 5: Real-Life Scenarios
1. curl http://myapp.com:8080
Isme ye concepts use hote hain:
DNS se domain resolve hota hai
HTTP protocol use hota hai
Port 8080 access hota hai
TCP connection banta hai

2. Database Not Connecting (10.0.1.50:3306)
Main sabse pehle check karunga:
Server reachable hai ya nahi (ping)
MySQL running hai ya nahi
Port open hai ya nahi
Firewall rules

What I Learned (My Key Learnings)
DNS domain ko IP me convert karta hai.
CIDR aur subnetting network manage karna easy banata hai.
Ports services ko identify karne me help karte hain.

Conclusion
Aaj maine networking ke basic concepts practically seekhe.
Isse mujhe samajh aaya ki real servers kaise communicate karte hain.
Ye knowledge DevOps aur Cloud ke liye bahut important hai.

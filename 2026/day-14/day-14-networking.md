## 2️⃣ Hands-on Checklist

### Identity – IP Address
Command:
hostname -I

Observation:
The system IP address was identified successfully, confirming correct network configuration.

---

### Reachability – Ping Test
Target: google.com

Command:
ping -c 4 google.com

Observation:
The host was reachable with low latency and no packet loss, indicating stable connectivity.

---

### Path – Network Route
Command:
traceroute google.com
(or tracepath google.com)

Observation:
Multiple network hops were observed with no major timeouts, showing normal routing.

---

### Ports – Listening Services
Command:
ss -tulpn

Observation:
SSH service was listening on port 22, confirming remote access availability.

---

### Name Resolution – DNS Check
Command:
nslookup google.com

Observation:
Domain name resolved correctly to an IP address, indicating DNS is working.

---

### HTTP Check
Command:
curl -I https://google.com

Observation:
Received HTTP status code 200 OK, confirming successful web connectivity.

---

### Connections Snapshot
Command:
ss -an | head

## 3️⃣ Mini Task – Port Probe & Interpretation

### Selected Port
Port: 22 (SSH)

---

### Port Test
Command:
nc -zv localhost 22

Observation:
The SSH port was reachable, confirming that the service is running.

---

### Next Check If Failed
If the port was unreachable, the next step would be to check:
- Service status using systemctl
- Firewall rules

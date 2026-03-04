# 🔐 TLS Fingerprinting using Mercury
![Platform](https://img.shields.io/badge/Platform-Linux-blue) ![Protocol](https://img.shields.io/badge/Protocol-TLS-red) ![Domain](https://img.shields.io/badge/Domain-Network%20Security-green) ![Tool](https://img.shields.io/badge/Analysis-Mercury-orange)

This project demonstrates **TLS fingerprinting in an isolated Linux network namespace environment**.

The experiment captures TLS handshake traffic from different clients and extracts **TLS fingerprints** using the **Mercury framework**.

The goal is to show that **different TLS clients generate unique fingerprints even when connecting to the same server**.

---

# 📌 Network Topology

We create an isolated virtual network using **Linux network namespaces**:

```
red (client) ---- router (packet capture) ---- blue (TLS server)
10.0.1.2             veth-r1: 1.0.1.1              10.0.2.2
                     veth-r2: 1.0.2.1
```

The router namespace captures packets using **tcpdump**, and the captured PCAP files are analyzed using **Mercury** to extract TLS fingerprints.

---

# 🛠 Requirements

Run this experiment on **Linux or WSL Ubuntu**.

Install required tools:

```bash
sudo apt update

sudo apt install -y \
    iproute2 \
    tcpdump \
    curl \
    openssl \
    jq \
    python3 \
    python3-pip \
    git \
```

Install required packages:

```bash
sudo apt install build-essential \
libssl-dev \
zlib1g-dev
```

---

# 📥 Clone Repository

```bash
git clone https://github.com/Samyak05/tls-fingerprinting-mercury.git

cd tls-fingerprinting-mercury
```

---

# ⚙️ Build Mercury

Mercury is used to extract TLS fingerprints from PCAP files. Clone it from Mercury official repo

```bash
git clone https://github.com/cisco/mercury.git
```

Build mercury
```bash
cd mercury
./configure
make

cd ..
```

After compilation the binary will be:

```
mercury/src/mercury
```

---

# 🌐 Setup Network Namespaces

Run the setup script:

```bash
sudo bash setup.sh
```

This script creates:

```
Namespaces
-----------
red      → TLS client
router   → packet capture
blue     → TLS server
```

Verify:

```bash
ip netns list
```

Expected output:

```
red
router
blue
```

---

# 🔑 Start TLS Server

Inside the **blue namespace**, generate a self-signed certificate.

```bash
sudo ip netns exec blue openssl req -x509 -newkey rsa:2048 \
-keyout key.pem \
-out cert.pem \
-days 365 \
-nodes
```

Start the TLS server:

```bash
sudo ip netns exec blue openssl s_server \
-accept 4433 \
-cert cert.pem \
-key key.pem
```

Leave this terminal running.

---

# 📡 Capture TLS Traffic

In another terminal, capture packets from the router namespace.

Example:

```bash
sudo ip netns exec router tcpdump -i veth-r1 -w pcaps/curl_tls12.pcap
```

Stop capture using:

```
CTRL + C
```

---

# 🚀 Generate TLS Traffic

Open a new terminal and generate traffic from the **red namespace**.

### OpenSSL Client

```bash
sudo ip netns exec red openssl s_client -connect 10.0.2.2:4433
```

### Curl TLS 1.2

```bash
sudo ip netns exec red curl -k --tls-max 1.2 https://10.0.2.2:4433
```

### Curl TLS 1.3

```bash
sudo ip netns exec red curl -k --tlsv1.3 https://10.0.2.2:4433
```

### Python SSL Client

```bash
sudo ip netns exec red python3 tls_test.py
```

For each client run, capture a separate PCAP file.

Generated captures:

```
pcaps/
├── curl_tls12.pcap
├── curl_tls13.pcap
├── python_ssl.pcap
└── openssl.pcap
```

---

# 🔍 Extract TLS Fingerprints

Run Mercury on each PCAP file:

```bash
for f in pcaps/*.pcap; do
  echo "===== $f ====="
  ./mercury/src/mercury -r "$f" | jq 'select(.fingerprints.tls)'
  echo
done
```

Example output:

```
tls/(0303)(130213031301...)
```

Structure:

```
tls/(TLS_VERSION)(CIPHER_SUITES)(EXTENSIONS)
```

---

# 📊 Observations

Different TLS clients produce different fingerprints due to:

- Cipher suite ordering
- TLS extensions
- TLS version support
- Library implementation

Example comparison:

| Client | TLS Version | Characteristics |
|------|------|------|
| curl TLS1.2 | TLS 1.2 | long cipher suite list |
| curl TLS1.3 | TLS 1.3 | modern cipher suites |
| Python SSL | TLS 1.3 | different extension ordering |
| OpenSSL | TLS 1.2 | minimal handshake |

---

# 📂 Repository Structure

```
tls-fingerprinting/
│
├── mercury/                 # Mercury fingerprinting framework
│
├── pcaps/                   # Captured TLS traffic
│   ├── curl_tls12.pcap
│   ├── curl_tls13.pcap
│   ├── python_ssl.pcap
│   └── openssl.pcap   
│
├── diagrams/
│   └── tls_fingerprint_comparison.png
│
├── results/
│   └── captured_fingerprints.json
│
├── scripts/
│   ├── tls_test.py              # Python TLS client
│   └── python_requests.py
│
├── setup.sh                 # Network namespace setup
│
├── README.md
└── .gitignore
```

---

## ⚠ Security Disclaimer

Self-signed certificates were used strictly for academic experimentation.

In production environments:

- Certificates must be issued by trusted Certificate Authorities (CA)
- TLS 1.3 is strongly recommended

---

# 📖 Key Learning

TLS fingerprinting works because **ClientHello messages differ across implementations**.

Even with encrypted traffic, the handshake metadata reveals:

- supported cipher suites
- extensions
- protocol versions

These differences allow **passive identification of TLS clients**.

---

# 📚 References

- [Cisco Mercury Documentation](https://github.com/cisco/mercury.git)
- [Wikipedia](https://share.google/dW41EQIiRAQRkYrfB)
- [TLS Protocol Specification (RFC 8446)](https://share.google/cqYXYOMIqCj5D5E6g)

---

# 👨‍💻 Author

Samyak Gedam 
M.Tech
National Institute of Technology Surathkal, Karnataka.
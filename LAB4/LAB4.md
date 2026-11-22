# Lab 4 — 5G CU/DU Split Deployment and F1-C/F1-U Analysis


**Environment:** Ubuntu 24.04 LTS · OpenAirInterface (OAI) v2.4.0 · RFsimulator · no SDR required 
**Prerequisite:** Lab 3 completed successfully (OAI CN5G, gNB, nrUE, UE tunnel and Ping)

| Parameter | Classroom configuration |
|---|---|
| OAI version | `v2.4.0` |
| RAN | One gNB-CU + one gNB-DU + one nrUE |
| Radio | RFsimulator, no physical SDR |
| NR band | n78 |
| Bandwidth | 106 PRBs |
| Numerology | 1 (30 kHz SCS) |
| CU F1 address (observed) | `127.0.0.3` |
| DU F1 address (observed) | `127.0.0.4` |
| DU SSB NR-ARFCN (observed) | `630048` (3,450.72 MHz) |
| UE frequency argument used in successful split test | `-C 3450720000` |
| UE IMSI | `001010000000001` |
| External DN | `192.168.70.135` |

**Important:** The Lab 3 monolithic gNB config used `absoluteFrequencySSB = 641280` (3,619.20 MHz), whereas the split DU config used `630048` (3,450.72 MHz). The frequency parameters **must match the working DU/UE pair**. Do not reuse Lab 3's `-C 3619200000` without checking the DU configuration. The `-C` argument and effective radio carrier placement should be verified against OAI startup logs; the above value is the one used in the successful classroom test, not a general formula for every DU file.

---

## Task 1 — Inspect the split configurations

### Step 1. Confirm the configuration files exist

```bash
cd ~/openairinterface5g-v2.4.0

ls -l \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf
```

### Step 2. Check frequency and F1 network settings

```bash
grep -nE \
'absoluteFrequencySSB|dl_absoluteFrequencyPointA|dl_carrierBandwidth|physCellId|min_rxtxtime|local_s_address|remote_s_address|local_n_address|remote_n_address|local_s_portd|remote_s_portd|local_n_portd|remote_n_portd|amf_ip_address' \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf
```

**Record:** The CU and DU F1 addresses, the DU SSB frequency, and PRB count. If the local settings differ from the table above, adapt the following commands and capture filters accordingly.

![image](LAB4/Image/1.jpg)

> CU F1 addresses: 127.0.0.3
> DU F1 addresses: 127.0.0.4
> DU SSB frequency: NR-ARFCN 630048 (3000 + 0.015 × 30048 = 3450.72 MHz)
> PRB count: 106 PRBs

**Questions**

1. Which protocol layers mainly reside in the CU, and which reside in the DU? 
> **Ans**: CU: RRC, SDAP, PDCP.  DU: RLC, MAC, PHY.
2. What is the purpose of the F1 interface?
> **Ans**: F1 connects the gNB-CU and gNB-DU. F1-C exchanges RAN control signaling using F1AP/SCTP, while F1-U transports UE user-plane data using GTP-U/UDP.
3. Why must the UE radio frequency be compatible with the DU rather than taken blindly from the previous lab?
> **Ans**: The frequency parameters must match the working DU/UE pair. The DU uses 630048 (3450.72 MHz), not Lab 3's 641280 (3619.20 MHz); otherwise the UE fails to decode SIB1.
---

## Task 2 — Deploy the split 5G SA network

Before starting, stop the monolithic Lab 3 gNB and any old nrUE processes with `Ctrl+C`. Avoid running two gNB/DU RFsimulator servers on the same listening port.

### Terminal 1 — Start CN5G

```bash
cd ~/oai-cn5g
docker compose up -d
docker compose ps
```

### Terminal 2 — Start gNB-CU

```bash
cd ~/openairinterface5g-v2.4.0/cmake_targets/ran_build/build

sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf
```

### Terminal 3 — Start gNB-DU

**Start the packet capture in Task 3 *before* starting the DU if you want to record the initial F1 Setup handshake.**

```bash
cd ~/openairinterface5g-v2.4.0/cmake_targets/ran_build/build

sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf \
  --rfsim
```

### Terminal 4 — Start nrUE

```bash
cd ~/openairinterface5g-v2.4.0/cmake_targets/ran_build/build

sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3450720000 \
  --uicc0.imsi 001010000000001 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1
```

> **Instructor note:** This UE command reflects the working classroom split configuration. In particular, the original Lab 3 `-C 3619200000` setting was not compatible with this DU configuration. If a provided classroom image has different NR frequencies or UICC credentials, use the instructor-validated values.

### Check the UE connection

```bash
ip -4 addr show oaitun_ue1
ping -I oaitun_ue1 -c 10 192.168.70.135
```

**Evidence:** Screenshots of CU/DU startup, UE connection, and successful Ping. Record the observed UE IP, transmitted/received packets, packet loss, and average RTT. Do **not** invent values; they vary by machine.

![image](Image/2.png)


![image](Image/3.png)


![image](Image/4.png)


![image](Image/5.png)

> The IMSI 001010000000099 row is left over from Lab 3.

![image](Image/6.png)


> UE IP: 10.0.0.2
> Transmitted / Received: 10 / 10
> Packet loss: 0%
> Average RTT: 5.125 ms

---

## Task 3 — Capture F1-C and F1-U in a single PCAP

The classroom capture showed CU `127.0.0.3` and DU `127.0.0.4` on the Linux loopback interface (`lo`).

### Step 1. Start packet capture

Start CN5G and CU, but **before starting the DU** open another terminal:

```bash
sudo tcpdump -i lo -s 0 -nn \
  -w ~/lab4-f1-complete.pcap \
  'host 127.0.0.3 and host 127.0.0.4 and (sctp or udp port 2152)'
```

This captures:

- **F1-C:** F1AP carried over SCTP.
- **F1-U:** GTP-U carried over UDP port 2152, **provided the configured F1-U endpoints and port match these values**.

### Step 2. Trigger control-plane and user-plane traffic

1. Start the DU to trigger **SCTP association** and **F1 Setup**.
2. Start nrUE to trigger **RRC** and **UE Context** procedures.
3. After the UE tunnel appears, generate traffic:

```bash
ping -I oaitun_ue1 -c 30 -i 0.2 192.168.70.135
```

4. Press `Ctrl+C` in the **tcpdump terminal** to finish and save the PCAP.
5. Verify the file:

```bash
ls -lh ~/lab4-f1-complete.pcap
```

### Alternative: capture all traffic between CU and DU

If you are unsure of the F1-U UDP port, use this **instead of** the filtered capture above:

```bash
sudo tcpdump -i lo -s 0 -nn \
  -w ~/lab4-f1-all.pcap \
  'host 127.0.0.3 and host 127.0.0.4'
```

If the CU and DU are not communicating over `lo`, select the correct interface (`ip -br addr` or `tcpdump -D`) and update the IP filter.

---

## Task 4 — Analyze F1-C (F1AP / SCTP)

Open the capture:

```bash
wireshark ~/lab4-f1-complete.pcap
```

Useful **Wireshark display filters**:

```text
sctp
f1ap
```

Find and record these messages:

| Message | Typical direction | Meaning |
|---|---|---|
| SCTP INIT / INIT_ACK / COOKIE_ECHO / COOKIE_ACK | DU ↔ CU | Establish SCTP association |
| F1 Setup Request | DU → CU | DU initiates F1 connection and provides served-cell information |
| F1 Setup Response | CU → DU | CU accepts F1 Setup |
| GNBDUConfigurationUpdate | DU → CU | DU informs CU of configuration changes |
| Initial UL RRC Message Transfer (RRC Setup Request) | DU → CU | UE initial RRC request forwarded to CU |
| DL RRC Message Transfer (RRC Setup) | CU → DU | CU delivers RRC Setup toward UE |
| UL RRC Message Transfer (RRC Setup Complete) | DU → CU | UE completes RRC procedure; NAS Registration Request can be carried inside |
| UE Context Setup Request / Response | CU ↔ DU | Establish UE-specific resources/context |
| DL RRC Message Transfer (RRC Reconfiguration) | CU → DU | CU sends further RRC configuration |

**Observed classroom sequence:** The successful capture contained SCTP association, F1 Setup Request/Response, DU Configuration Update/Acknowledgement, RRC Setup messages, Authentication signaling carried through RRC, UE Context Setup/Modification, and RRC Reconfiguration. The exact packet numbers and timing are **not fixed**.

**Questions**

1. Which node initiates F1 Setup, and which acknowledges it?
> **Ans**: The DU initiates (F1 Setup Request). The CU acknowledges (F1 Setup Response).
2. Explain the difference between SCTP association establishment and F1 Setup.
> **Ans**: The SCTP association is the transport-layer connection. F1 Setup is the F1AP application-layer procedure carried over it.
3. How can a NAS Registration Request be transported over F1-C?
> **Ans**: The NAS Registration Request is carried inside RRC Setup Complete, which the DU forwards to the CU in a UL RRC Message Transfer.
4. What is the role of UE Context Setup?
> **Ans**: It establishes UE-specific resources/context (SRB/DRB) in the DU.
5. Does F1 Setup alone prove successful UE Registration? Explain.
> **Ans**: No. F1 Setup only proves the CU–DU connection. UE Registration and end-to-end connectivity must be validated separately, for example by the AMF showing the UE as 5GMM-REGISTERED.

**Evidence:** Screenshot with F1 Setup Request/Response and screenshot with at least three UE-associated F1AP messages. Include packet numbers and source/destination addresses.

f1ap.png
![截圖 2026-11-22 下午3.54.48](https://hackmd.io/_uploads/SkoA17ly7l.jpg)

| Frame | Src → Dst | Message |
|---|---|---|
| 1–4 | 127.0.0.4 ↔ 127.0.0.3 | SCTP INIT / INIT_ACK / COOKIE_ECHO / COOKIE_ACK |
| 5 | 127.0.0.4 → 127.0.0.3 | F1SetupRequest |
| 7 | 127.0.0.3 → 127.0.0.4 | F1SetupResponse |
| 9 | 127.0.0.4 → 127.0.0.3 | GNBDUConfigurationUpdate |
| 12 | 127.0.0.4 → 127.0.0.3 | InitialULRRCMessageTransfer (RRC Setup Request) |
| 13 | 127.0.0.3 → 127.0.0.4 | DLRRCMessageTransfer (RRC Setup) |
| 14 | 127.0.0.4 → 127.0.0.3 | ULRRCMessageTransfer (RRC Setup Complete) |
| 27 / 28 | 127.0.0.3 ↔ 127.0.0.4 | UEContextSetupRequest / Response |
| 29 | 127.0.0.3 → 127.0.0.4 | DLRRCMessageTransfer (RRC Reconfiguration) |

**UE-associated F1AP messages**

| Frame | Message | Explanation |
|---|---|---|
| 12 | InitialULRRCMessageTransfer | UE initial RRC request forwarded to CU. |
| 13 | DLRRCMessageTransfer | CU delivers RRC Setup toward UE. |
| 14 | ULRRCMessageTransfer | UE completes RRC procedure; NAS Registration Request is carried inside. |


---

## Task 5 — Analyze F1-U (GTP-U / UDP)

In the **same PCAP**, apply:

```text
gtp
```

If necessary, first check:

```text
udp.port == 2152
```

Directional display filters (based on the observed classroom CU/DU IP pair):

```text
gtp && ip.src == 127.0.0.4 && ip.dst == 127.0.0.3
gtp && ip.src == 127.0.0.3 && ip.dst == 127.0.0.4
```

In the captured GTP-U packet details, inspect:

```text
Internet Protocol (outer CU/DU addresses)
  User Datagram Protocol
    GPRS Tunneling Protocol (GTP-U)
      Message Type: G-PDU / T-PDU
      Tunnel Endpoint Identifier (TEID)
      Encapsulated user payload (if decodable)
```

**Important:** The classroom trace showed repeated `GTP T-PDU` packets from `127.0.0.4` (DU) to `127.0.0.3` (CU), confirming traffic on the CU–DU user-plane path. The actual TEID was **not recorded** in the provided screenshot: read the TEID from your own packet details instead of assuming a value.

### Distinguish F1-U from N3

Both **F1-U** and **N3** can use GTP-U. Do **not** classify traffic as F1-U solely because Wireshark says `GTP` or the port is `2152`.

- **F1-U:** GTP-U between **gNB-DU and gNB-CU**.
- **N3:** GTP-U between **gNB-CU and 5GC UPF**.

Identify the interface using the actual source/destination endpoints and configuration.

| Interface | Endpoints | Protocol | Carries |
|---|---|---|---|
| F1-C | DU ↔ CU | F1AP / SCTP | F1 Setup, RRC transport, UE Context |
| F1-U | DU ↔ CU | GTP-U / UDP | UE traffic (internal split-RAN user-plane link) |
| N3 | CU ↔ UPF | GTP-U / UDP | UE traffic (5GC ↔ RAN user-plane link) |

**Questions**

1. Which transport protocol carries F1-U?
> **Ans**: UDP.
2. What does a GTP-U **TEID** identify?
> **Ans**: The GTP-U tunnel endpoint at the receiver, which tells it which UE bearer the packet belongs to.
3. Which direction is `127.0.0.4 → 127.0.0.3` in this setup?
> **Ans**: Uplink (DU → CU)
4. How would you distinguish F1-U from N3 in a packet capture?
> **Ans**: By the outer source/destination IPs. F1-U is GTP-U between gNB-DU and gNB-CU; N3 is GTP-U between gNB-CU and 5GC UPF.
5. Why do F1-C and F1-U use different transport protocols?
> **Ans**: F1-C signaling needs reliable, in-order delivery (SCTP). F1-U user data needs low overhead and low latency (UDP).

**Evidence:** Screenshot of at least one GTP-U T-PDU, with outer IP addresses, UDP ports, message type, and TEID visible. Record the observed UL and DL packet counts (if both directions are present). Do not assume equal counts.

![image](Image/7.png)


![image](Image/8.png)


![image](Image/9.png)


| Direction | Outer IP | UDP ports | Type | TEID | Count |
|---|---|---|---|---|---|
| UL | 127.0.0.4 → 127.0.0.3 | 2152 → 2152 | T-PDU | 0xb60080a8 | 44 |
| DL | 127.0.0.3 → 127.0.0.4 | 2152 → 2152 | T-PDU | 0x72b309b1 | 40 |


---

## Task 6 — Compare monolithic and split gNB

Complete the table:

| Feature | Lab 3: Monolithic gNB | Lab 4: CU/DU Split |
|---|---|---|
| Number of RAN processes |1|2 |
| Location of RRC/PDCP | gNB| CU|
| Location of RLC/MAC/PHY |gNB |DU |
| F1-C interface present? | No|Yes |
| F1-U interface present? | No| Yes|
| UE connected to the same 5GC? |Yes |Yes |
| End-to-end Ping works? | Yes| Yes|

Explain in **3–5 sentences** why splitting a gNB changes the internal RAN interfaces but does not change the UE's need for synchronization, registration and a PDU session.
> **Ans**: Splitting the gNB only changes the functional division between gNB-CU and gNB-DU, which adds the internal split-RAN interfaces F1-C and F1-U. To the UE, the CU and DU still act as one gNB with the same radio interface and the same RRC and NAS procedures. Therefore the UE still has to synchronize to the SSB, register with the AMF, and establish a PDU session.


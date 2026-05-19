# Proxmox VE & pfSense Home Gateway

🌐 **English** | [🇮🇹 Vai alla versione italiana ↓](#-versione-italiana)

> **Virtualized firewall/gateway appliance** built on Proxmox VE with a pfSense VM acting as the sole network gateway. This repository documents the architectural evolution, the Layer 1 troubleshooting performed, and the final production configuration capable of sustaining symmetric Gigabit line-rate.

![Status](https://img.shields.io/badge/status-production-brightgreen)
![Proxmox](https://img.shields.io/badge/Proxmox%20VE-8.x%2F9.x-orange)
![pfSense](https://img.shields.io/badge/pfSense-CE-blue)
![Throughput](https://img.shields.io/badge/Throughput-940%2F948%20Mbps-success)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Host Hardware](#2-host-hardware)
3. [Architecture Evolution](#3-architecture-evolution)
4. [Diagnostic Analysis and Root Cause](#4-diagnostic-analysis-and-root-cause)
5. [Final Production Architecture](#5-final-production-architecture)
6. [Proxmox Network Configuration](#6-proxmox-network-configuration)
7. [pfSense VM Optimizations](#7-pfsense-vm-optimizations)
8. [Benchmark Results](#8-benchmark-results)
9. [Lessons Learned](#9-lessons-learned)

---

## 1. Project Overview

The goal of this project was to migrate from a consumer-grade home networking setup to a **virtualized routing infrastructure** providing low-latency, high-throughput packet forwarding on a symmetric Gigabit fiber line, while keeping the hardware footprint minimal and power consumption suitable for 24/7 operation.

The chosen platform consists of:

- **Proxmox VE** as the type-1 hypervisor running on a low-power x86 mini-PC.
- **pfSense (FreeBSD-based)** as a dedicated VM acting as the network's single gateway, performing routing, filtering, and packet inspection.

The project is intentionally **infrastructure-focused**: it documents how the gateway was built and stabilized, including the hardware iteration that was required to reach line-rate throughput.

---

## 2. Host Hardware

The system runs on an x86 mini-PC form factor optimized for continuous H24 operation. Hardware virtualization extensions (**Intel VT-x and VT-d**) are enabled at the BIOS/UEFI level to ensure correct logical pass-through of resources to the guest VMs.

| Component | Specification |
|---|---|
| **Form factor** | Industrial / consumer compact Mini-PC |
| **Onboard NIC** | Native Gigabit Ethernet controller (assigned as `nic0`), 10/100/1000 Mbps theoretical support |
| **PCIe Expansion NIC** | **Intel i226-V 2.5 GbE**, installed via an M.2 (Key A+E / B+M) to PCIe riser module to attach the controller directly to the native PCI Express bus |
| **Downstream device** | Keenetic router/access point cascaded for Wi-Fi Mesh distribution and local switching (1 Gbps physical ports) |

The use of an M.2-to-PCIe riser allows the addition of a true PCI Express NIC even on a chassis that does not expose a traditional PCIe slot, avoiding any reliance on USB-based network adapters.

---

## 3. Architecture Evolution

### Phase 1 — Initial Implementation (USB Bottleneck)

The first iteration needed two distinct physical interfaces for pfSense (WAN and LAN). To satisfy this requirement, the onboard NIC was paired with a **USB-to-Ethernet adapter** (ASIX/Realtek-class chipset).

Initial logical topology:

```
[ISP Modem] ──► [USB-to-Ethernet (WAN)] ──► [pfSense VM] ──► [Onboard NIC (LAN)] ──► [Downstream Router]
```

This setup exhibited severe structural limitations:

- **USB bus polling overhead** introduced by the hypervisor.
- **FreeBSD kernel driver instability** for consumer-grade USB Ethernet chipsets.
- **Throughput consistently capped at ~90–94 Mbps**, regardless of the underlying link speed.

The cap was artificial and reproducible across multiple load tests, making it clear that the USB adapter was an unsuitable carrier for a gateway role.

### Phase 2 — Transition to a Dedicated PCI Express Bus

To eliminate USB-related instability, the **Intel i226-V PCIe** controller was introduced via the M.2 riser. In the first configuration of this phase, the **logical roles were kept unchanged**:

- Onboard NIC → WAN (connected to the ISP modem)
- New Intel i226-V → LAN (connected to the downstream router)

Despite the removal of the USB adapter, **bandwidth tests still showed a symmetric ceiling at 94 Mbps**, both from LAN clients and directly from the Proxmox host shell. This was a strong indicator of a **Layer 1 (physical layer) negotiation issue** rather than a software or driver bottleneck.

---

## 4. Diagnostic Analysis and Root Cause

Diagnosis was performed directly from the Proxmox host shell using the low-level `ethtool` utility against the onboard NIC.

Relevant fields from `ethtool` output:

```
Supported ports: [ TP ]
Supported link modes:   10baseT/Half 10baseT/Full
                        100baseT/Half 100baseT/Full
                        1000baseT/Full
Supports auto-negotiation: Yes

Advertised link modes:  10baseT/Half 10baseT/Full
                        100baseT/Half 100baseT/Full
                        1000baseT/Full

Link partner advertised link modes:  10baseT/Half 10baseT/Full
                                     100baseT/Half 100baseT/Full
Speed: 100Mb/s
Duplex: Full
Auto-negotiation: on
```

### Technical Interpretation

The `Link partner advertised link modes` field **omitted `1000baseT/Full` entirely**. The host was advertising Gigabit capability correctly, but the upstream device refused to negotiate it.

The root cause is attributable to one or more of the following factors on the upstream side:

- **Restrictive energy profiles** (Green Mode / Eco Mode) implemented by the ISP-provided modem firmware.
- **Electrical impedance incompatibility** between the modem's PHY and the integrated controller on the host mainboard.

As a result, the link was forced down to **100 Mbps Fast Ethernet**, yielding ~94 Mbps of useful throughput once packet encapsulation overhead is subtracted.

Any attempt to **force the speed to 1000 Mbps Full Duplex via software** caused the immediate loss of the physical carrier (`Speed: Unknown`), confirming that the misalignment could not be resolved through driver-level configuration alone — the issue was rooted in the physical layer.

---

## 5. Final Production Architecture

Since replacing the ISP-provided modem was not an option, the solution was to **swap the logical roles of the two NICs**. The Intel i226-V controller — featuring **wider electrical tolerances and a server-class chipset fully compliant with IEEE 802.3ab standards** — was reassigned to the WAN, placed in direct contact with the ISP modem. The onboard NIC was moved to the LAN, where the downstream router's switch ASIC handles negotiation reliably even with legacy PHYs.

```
                        ┌─────────────────────────────┐
                        │      ISP Fiber Modem        │
                        └──────────────┬──────────────┘
                                       │
                       Cat6 — Negotiation OK @ 1.0 Gbps
                                       │
                                       ▼
                  ┌─────────────────────────────────────┐
                  │   Intel i226-V PCIe — WAN (new)     │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │  Proxmox VE Host  ──►  pfSense VM   │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │     Onboard NIC — LAN (new)         │
                  └──────────────────┬──────────────────┘
                                     │
                              Cat6 @ 1.0 Gbps
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │     Keenetic Router / AP            │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │           LAN Clients               │
                  └─────────────────────────────────────┘
```

The inversion is purely **logical and electrical** — no additional hardware was added. Both links stabilized immediately at **1.0 Gbps Full Duplex** after the swap.

---

## 6. Proxmox Network Configuration

The host network configuration (`/etc/network/interfaces`) defines two Linux bridges, each anchored to one physical interface. pfSense receives both bridges as its WAN and LAN ports.

```ini
auto lo
iface lo inet loopback

# Intel i226-V PCI Express controller — assigned to WAN
iface enp2s0 inet manual

# Onboard NIC — assigned to LAN
iface nic0 inet manual

# WAN bridge — bound to the PCIe controller (high-tolerance link)
auto vmbr0
iface vmbr0 inet static
    address  <management-address>/24
    gateway  <upstream-gateway>
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0

# LAN bridge — bound to the onboard NIC
auto vmbr1
iface vmbr1 inet manual
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
```

> Replace `<management-address>` and `<upstream-gateway>` with values appropriate for your environment. The management address is used to reach the Proxmox Web UI from the WAN-side subnet; depending on the security model adopted, the host management interface can also be moved entirely behind the LAN bridge.

Interface names (`enp2s0`, `nic0`) will vary depending on kernel naming and the specific hardware — use `ip link` to confirm which name maps to which physical port.

---

## 7. pfSense VM Optimizations

To prevent CPU saturation caused by software packet emulation and to avoid offload conflicts between the host bridges and the guest, the following settings are applied inside the pfSense VM.

### 7.1 Paravirtualized NICs

The virtual NICs attached to the pfSense VM are configured as **VirtIO (paravirtualized)**. Heavier emulated models such as Intel E1000 are explicitly avoided, since they consume significantly more CPU cycles per packet on KVM and cap throughput well below line-rate.

### 7.2 Hardware Offloading Disabled

Under **System → Advanced → Networking** in the pfSense Web UI, the following flags are enabled (which disables the corresponding offload):

- ✅ **Disable hardware checksum offload**
- ✅ **Disable hardware TCP segmentation offload**
- ✅ **Disable hardware large receive offload**

These offloads must be delegated to the VirtIO engine to avoid checksum mismatches and dropped packets at the Linux bridge layer. Leaving them enabled is a common source of subtle packet loss and degraded TCP throughput in Proxmox + pfSense deployments.

---

## 8. Benchmark Results

After applying the new physical and logical layout, line-rate symmetric Gigabit was achieved.

| Metric | Result |
|---|---|
| **Download** | 940.34 Mbps |
| **Upload** | 948.12 Mbps |
| **Packet loss** | 0.0% |
| **Latency** | 2.01 ms |
| **Jitter** | 0.1 ms |
| **Server** | Milan (Extra-Hop Node) |

Measurements were performed using the Ookla Speedtest CLI from a wired LAN client. The link saturates the contractual 1 Gbps fiber profile in both directions, with negligible latency overhead introduced by the virtualized firewall.

---

## 9. Lessons Learned

The most important takeaway from this project:

> **Layer 1 problems cannot be solved at Layer 2 or above.**

Several days of driver tuning, MTU adjustments, and offload toggling did not move the needle as long as the underlying PHY-to-PHY negotiation was failing. The fix required physically reassigning hardware roles based on each controller's electrical tolerance and standards compliance, not software changes.

Other practical takeaways:

- **USB Ethernet adapters are unsuitable for production gateway roles** under FreeBSD-based firewalls. The combination of bus polling overhead and driver immaturity caps throughput well below Gigabit even on otherwise capable hardware.
- **`ethtool` is the single most useful diagnostic tool** when investigating throughput anomalies. The `Link partner advertised link modes` field is often the fastest way to confirm whether a negotiation problem originates upstream or downstream.
- **Server-class NIC controllers (Intel i225/i226 series) tolerate non-ideal PHY conditions** far better than embedded consumer chipsets, making them the right choice for the link directly facing ISP equipment.
- **VirtIO + disabled hardware offload is the canonical pfSense-on-KVM tuning pair**. Skipping it produces hard-to-diagnose throughput regressions even when the physical layer is healthy.

---

## License

This documentation is provided as-is for reference and educational purposes.

---
---

<a name="-versione-italiana"></a>

# 🇮🇹 Versione Italiana

[⬆ Back to English](#proxmox-ve--pfsense-home-gateway)

> **Appliance firewall/gateway virtualizzata** basata su Proxmox VE con una VM pfSense come unico gateway di rete. Questo repository documenta l'evoluzione architetturale, il troubleshooting effettuato a livello fisico e la configurazione di produzione finale, capace di saturare in modo stabile una linea Gigabit simmetrica.

---

## Indice

1. [Panoramica del Progetto](#1-panoramica-del-progetto)
2. [Hardware dell'Host](#2-hardware-dellhost)
3. [Evoluzione dell'Architettura](#3-evoluzione-dellarchitettura)
4. [Analisi Diagnostica e Root Cause](#4-analisi-diagnostica-e-root-cause)
5. [Architettura Finale di Produzione](#5-architettura-finale-di-produzione)
6. [Configurazione di Rete Proxmox](#6-configurazione-di-rete-proxmox)
7. [Ottimizzazioni della VM pfSense](#7-ottimizzazioni-della-vm-pfsense)
8. [Risultati del Benchmark](#8-risultati-del-benchmark)
9. [Lezioni Apprese](#9-lezioni-apprese)

---

## 1. Panoramica del Progetto

L'obiettivo del progetto è la migrazione da un setup di rete domestica consumer a un'**infrastruttura di routing virtualizzata** a bassa latenza e alto throughput, capace di saturare una linea in fibra Gigabit simmetrica, mantenendo una footprint hardware ridotta e un consumo energetico adatto al funzionamento continuo H24.

La piattaforma scelta è composta da:

- **Proxmox VE** come hypervisor di tipo 1, eseguito su un mini-PC x86 a basso consumo.
- **pfSense (basato su FreeBSD)** come VM dedicata che funge da unico gateway della rete, gestendo routing, filtraggio e ispezione dei pacchetti.

Il progetto è volutamente **focalizzato sull'infrastruttura**: documenta come il gateway è stato costruito e stabilizzato, inclusa l'iterazione hardware che si è resa necessaria per raggiungere il line-rate.

---

## 2. Hardware dell'Host

Il sistema si basa su un mini-PC x86 ottimizzato per il funzionamento continuo H24. Le estensioni hardware per la virtualizzazione (**Intel VT-x e VT-d**) sono abilitate a livello di BIOS/UEFI per garantire il corretto pass-through logico delle risorse verso le VM guest.

| Componente | Specifica |
|---|---|
| **Fattore di forma** | Mini-PC industriale / consumer compatto |
| **NIC Integrata** | Controller Gigabit Ethernet nativo (assegnato come `nic0`), supporto teorico 10/100/1000 Mbps |
| **NIC di Espansione PCIe** | **Intel i226-V 2.5 GbE**, installata tramite modulo riser M.2 (Key A+E / B+M) a PCIe per attestare il controller direttamente sul bus PCI Express nativo |
| **Dispositivo Downstream** | Router/Access Point Keenetic collegato in cascata per la distribuzione Wi-Fi Mesh e lo switching locale (porte fisiche a 1 Gbps) |

L'utilizzo di un riser M.2-to-PCIe permette di aggiungere una vera NIC PCI Express anche su uno chassis che non espone slot PCIe tradizionali, evitando qualsiasi dipendenza da adattatori di rete USB.

---

## 3. Evoluzione dell'Architettura

### Fase 1 — Implementazione Iniziale (Collo di Bottiglia USB)

La prima iterazione richiedeva due interfacce fisiche distinte per pfSense (WAN e LAN). Per soddisfare questo requisito, la NIC integrata è stata affiancata da un **adattatore USB-to-Ethernet** (chipset ASIX/Realtek).

Topologia logica iniziale:

```
[Modem ISP] ──► [USB-to-Ethernet (WAN)] ──► [VM pfSense] ──► [NIC Integrata (LAN)] ──► [Router Downstream]
```

Questo assetto ha manifestato severe limitazioni strutturali:

- **Overhead del polling sul bus USB** introdotto dall'hypervisor.
- **Instabilità dei driver FreeBSD** per chipset USB Ethernet di fascia consumer.
- **Throughput costantemente bloccato a ~90–94 Mbps**, indipendentemente dalla velocità del link sottostante.

Il blocco era artificiale e riproducibile attraverso molteplici test di carico, rendendo evidente che l'adattatore USB non era un supporto adeguato per un ruolo di gateway.

### Fase 2 — Transizione a Bus PCI Express Dedicato

Per eliminare l'instabilità legata all'USB, è stato introdotto il controller **Intel i226-V PCIe** tramite il riser M.2. Nella prima configurazione di questa fase, i **ruoli logici sono stati mantenuti invariati**:

- NIC Integrata → WAN (collegata al modem ISP)
- Nuova Intel i226-V → LAN (collegata al router downstream)

Nonostante la rimozione dell'adattatore USB, **i test di banda mostravano ancora un tetto simmetrico a 94 Mbps**, sia dai client LAN sia direttamente dalla shell dell'host Proxmox. Questo è un sintomo classico di un **problema di negoziazione a Layer 1 (livello fisico)**, e non di un collo di bottiglia software o driver.

---

## 4. Analisi Diagnostica e Root Cause

La diagnosi è stata effettuata direttamente dalla shell dell'host Proxmox utilizzando l'utility low-level `ethtool` sulla NIC integrata.

Campi rilevanti dall'output di `ethtool`:

```
Supported ports: [ TP ]
Supported link modes:   10baseT/Half 10baseT/Full
                        100baseT/Half 100baseT/Full
                        1000baseT/Full
Supports auto-negotiation: Yes

Advertised link modes:  10baseT/Half 10baseT/Full
                        100baseT/Half 100baseT/Full
                        1000baseT/Full

Link partner advertised link modes:  10baseT/Half 10baseT/Full
                                     100baseT/Half 100baseT/Full
Speed: 100Mb/s
Duplex: Full
Auto-negotiation: on
```

### Interpretazione Tecnica

Il campo `Link partner advertised link modes` **ometteva completamente `1000baseT/Full`**. L'host stava pubblicizzando correttamente la capacità Gigabit, ma il dispositivo upstream si rifiutava di negoziarla.

La causa principale è attribuibile a uno o più dei seguenti fattori sul lato upstream:

- **Profili energetici restrittivi** (Green Mode / Eco Mode) implementati dal firmware del modem ISP.
- **Incompatibilità di impedenza elettrica** tra il PHY del modem e il controller integrato sulla scheda madre dell'host.

Di conseguenza, il link veniva forzato a **100 Mbps Fast Ethernet**, ottenendo ~94 Mbps di throughput utile al netto dell'overhead di incapsulamento dei pacchetti.

Qualsiasi tentativo di **forzare la velocità a 1000 Mbps Full Duplex via software** causava la caduta immediata della portante fisica (`Speed: Unknown`), confermando che il disallineamento non poteva essere risolto attraverso la sola configurazione a livello driver — il problema era radicato nel livello fisico.

---

## 5. Architettura Finale di Produzione

Dal momento che sostituire il modem fornito dall'ISP non era un'opzione, la soluzione è consistita nello **scambiare i ruoli logici delle due NIC**. Il controller Intel i226-V — caratterizzato da **tolleranze elettriche superiori e un chipset di classe server pienamente conforme allo standard IEEE 802.3ab** — è stato riassegnato alla WAN, posto in contatto diretto con il modem ISP. La NIC integrata è stata spostata sulla LAN, dove l'ASIC dello switch del router downstream gestisce la negoziazione in modo affidabile anche con PHY legacy.

```
                        ┌─────────────────────────────┐
                        │      Modem Fibra ISP        │
                        └──────────────┬──────────────┘
                                       │
                       Cat6 — Negoziazione OK @ 1.0 Gbps
                                       │
                                       ▼
                  ┌─────────────────────────────────────┐
                  │   Intel i226-V PCIe — WAN (nuova)   │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │  Host Proxmox VE  ──►  VM pfSense   │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │     NIC Integrata — LAN (nuova)     │
                  └──────────────────┬──────────────────┘
                                     │
                              Cat6 @ 1.0 Gbps
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │     Router/AP Keenetic              │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │           Client LAN                │
                  └─────────────────────────────────────┘
```

L'inversione è puramente **logica ed elettrica** — non è stato aggiunto alcun hardware. Entrambi i link si sono stabilizzati immediatamente a **1.0 Gbps Full Duplex** dopo lo scambio.

---

## 6. Configurazione di Rete Proxmox

Il file di configurazione di rete dell'host (`/etc/network/interfaces`) definisce due bridge Linux, ciascuno ancorato a un'interfaccia fisica. pfSense riceve entrambi i bridge come proprie porte WAN e LAN.

```ini
auto lo
iface lo inet loopback

# Controller Intel i226-V PCI Express — assegnato alla WAN
iface enp2s0 inet manual

# NIC Integrata — assegnata alla LAN
iface nic0 inet manual

# Bridge WAN — collegato al controller PCIe (link ad alta tolleranza)
auto vmbr0
iface vmbr0 inet static
    address  <indirizzo-management>/24
    gateway  <gateway-upstream>
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0

# Bridge LAN — collegato alla NIC integrata
auto vmbr1
iface vmbr1 inet manual
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
```

> Sostituire `<indirizzo-management>` e `<gateway-upstream>` con i valori appropriati per il proprio ambiente. L'indirizzo di management viene utilizzato per raggiungere la Web UI di Proxmox dalla subnet lato WAN; a seconda del modello di sicurezza adottato, l'interfaccia di management dell'host può anche essere spostata interamente dietro il bridge LAN.

I nomi delle interfacce (`enp2s0`, `nic0`) varieranno in base al naming del kernel e all'hardware specifico — usare `ip link` per verificare la corrispondenza tra nome logico e porta fisica.

---

## 7. Ottimizzazioni della VM pfSense

Per prevenire la saturazione della CPU causata dall'emulazione software dei pacchetti ed evitare conflitti di offload tra i bridge dell'host e il guest, vengono applicate le seguenti configurazioni all'interno della VM pfSense.

### 7.1 NIC Paravirtualizzate

Le NIC virtuali collegate alla VM pfSense sono configurate come **VirtIO (paravirtualizzate)**. I modelli emulati più pesanti, come Intel E1000, vengono esplicitamente evitati, poiché consumano significativamente più cicli CPU per pacchetto su KVM e limitano il throughput ben al di sotto del line-rate.

### 7.2 Hardware Offloading Disattivato

Nel menu **System → Advanced → Networking** della Web UI di pfSense vengono attivati i seguenti flag (che disabilitano l'offload corrispondente):

- ✅ **Disable hardware checksum offload**
- ✅ **Disable hardware TCP segmentation offload**
- ✅ **Disable hardware large receive offload**

Questi offload devono essere delegati al motore VirtIO per evitare mismatch di checksum e pacchetti scartati a livello dei bridge Linux. Lasciarli attivi è una causa frequente di sottile packet loss e di throughput TCP degradato nei deployment Proxmox + pfSense.

---

## 8. Risultati del Benchmark

Dopo aver applicato il nuovo layout fisico e logico, il line-rate Gigabit simmetrico è stato raggiunto stabilmente.

| Metrica | Risultato |
|---|---|
| **Download** | 940.34 Mbps |
| **Upload** | 948.12 Mbps |
| **Packet loss** | 0.0% |
| **Latenza** | 2.01 ms |
| **Jitter** | 0.1 ms |
| **Server** | Milano (Extra-Hop Node) |

Le misurazioni sono state effettuate con la CLI di Ookla Speedtest da un client LAN cablato. Il link satura il profilo contrattuale della fibra a 1 Gbps in entrambe le direzioni, con un overhead di latenza trascurabile introdotto dal firewall virtualizzato.

---

## 9. Lezioni Apprese

Il takeaway più importante di questo progetto:

> **I problemi di Layer 1 non possono essere risolti a Layer 2 o superiore.**

Diversi giorni di tuning dei driver, aggiustamenti dell'MTU e switching dei flag di offload non hanno spostato di un millimetro l'ago della bilancia finché la negoziazione PHY-to-PHY sottostante stava fallendo. La soluzione ha richiesto la riassegnazione fisica dei ruoli hardware in base alla tolleranza elettrica e alla conformità agli standard di ciascun controller, non modifiche software.

Altri takeaway pratici:

- **Gli adattatori USB Ethernet non sono adatti a ruoli di gateway in produzione** sotto firewall basati su FreeBSD. La combinazione di overhead del polling sul bus e immaturità dei driver limita il throughput ben al di sotto del Gigabit anche su hardware altrimenti capace.
- **`ethtool` è lo strumento diagnostico più utile in assoluto** quando si investigano anomalie di throughput. Il campo `Link partner advertised link modes` è spesso il modo più rapido per confermare se un problema di negoziazione abbia origine upstream o downstream.
- **I controller NIC di classe server (serie Intel i225/i226) tollerano condizioni PHY non ideali** molto meglio dei chipset consumer embedded, rendendoli la scelta corretta per il link a diretto contatto con l'apparato dell'operatore.
- **VirtIO + hardware offload disabilitato è la coppia di tuning canonica per pfSense-on-KVM**. Saltarla produce regressioni di throughput difficili da diagnosticare anche quando il livello fisico è in salute.

---

## Licenza

Questa documentazione è fornita "as-is" a scopo di riferimento ed educativo.

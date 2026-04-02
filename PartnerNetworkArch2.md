# Mission‑critical extranet F5 GTM / LTM / WAF architecture

_Version: 1.0_  
_Date: 2026‑04‑02_

---

## 1. Architecture overview

Target state:

- Two sites: **Primary** and **DR**.  
- Each site has:
  - **2+1 F5 BIG‑IP DNS (GTM)** – 2‑node cluster + 1 independent hot‑spare node.  
  - **2+1 F5 BIG‑IP LTM** – 2‑node cluster + 1 hot‑spare node.  
  - **2+1 F5 BIG‑IP WAF** – 2‑node cluster + 1 hot‑spare node, behind LTM.  
- **Private cloud** spans both sites:
  - Virtual cloud firewall (CFW) in front.  
  - Virtual load‑balancer (ELB) in front of apps.  
- Some apps: **active/active**, some **active/passive**.  
- Some apps need to call back into the extranet via DNS/GTM.  
- **External clients** connect via **private leased lines** (not public internet).  
- **Corporate internal clients** resolve via **corporate intranet DNS** that delegates the extranet domain to GTM‑managed DNS.

---

## 2. Mermaid architecture diagram

> **Note for GitHub**: native GitHub Markdown does not render Mermaid.  
> To view this diagram, use:
> - VS Code with Mermaid plugin  
> - GitLab or another Mermaid‑enabled renderer  
> - Or an online Mermaid live editor.

``` mermaid
***
title: Mission‑critical extranet with F5 GTM / LTM / WAF and private cloud
***
flowchart TB
    subgraph "External Partners (Private leased lines)"
        A1["Partner Site A\n(Client)"]
        A2["Partner Site B\n(Client)"]
    end

    subgraph "Corporate Internal Clients"
        B1["Corp Internal Client\n(Windows/Linux)"]
        B2["Corporate Intranet DNS\n(forwards extranet domain)"]
    end

    subgraph "Primary Site"
        C1["GTM/DNS Cluster 1\n(Primary DNS cluster, 2 nodes)"]
        C2["GTM/DNS Spare 1\n(Independent hot‑spare node)"]
        C3["LTM Cluster 1\n(2 nodes)"]
        C4["LTM Spare 1\n(Hot‑spare node)"]
        C5["WAF Cluster 1\n(2 nodes)"]
        C6["WAF Spare 1\n(Hot‑spare node)"]
    end

    subgraph "DR Site"
        D1["GTM/DNS Cluster 2\n(Primary DNS cluster, 2 nodes)"]
        D2["GTM/DNS Spare 2\n(Independent hot‑spare node)"]
        D3["LTM Cluster 2\n(2 nodes)"]
        D4["LTM Spare 2\n(Hot‑spare node)"]
        D5["WAF Cluster 2\n(2 nodes)"]
        D6["WAF Spare 2\n(Hot‑spare node)"]
    end

    subgraph "Private Cloud (Site‑A)"
        E1["Virtual Cloud Firewall (CFW) A"]
        E2["Virtual Load‑Balancer (ELB A)"]
        E3["App Server A1 (Active)"]
        E4["App Server A2 (Active/Passive)"]
    end

    subgraph "Private Cloud (Site‑B)"
        F1["Virtual Cloud Firewall (CFW) B"]
        F2["Virtual Load‑Balancer (ELB B)"]
        F3["App Server B1 (Active)"]
        F4["App Server B2 (Active/Passive)"]
    end

    %% External → DNS (1–4)
    A1 -->|"GTM‑managed DNS query 1"| C1
    A2 -->|"GTM‑managed DNS query 2"| D1
    B1 -->|"Query intranet DNS 3"| B2
    B1 -->|"Query intranet DNS 4"| B2
    B2 -->|"Delegates to GTM 5"| C1
    B2 -->|"Delegates to GTM 6"| D1

    %% DNS → GTM site priority (7–10)
    C1 -->|"Local DNS zone (GTM) 7"| C1
    C1 -->|"Topology / site priority 8"| C3
    D1 -->|"Local DNS zone (GTM) 9"| D1
    D1 -->|"Topology / site priority 10"| D3

    %% LTM → WAF → Private Cloud (11–19)
    C3 -->|"HTTP(S) traffic 11"| C5
    C3 -->|"Health/management 12"| C4
    C5 -->|"Allowed traffic 13"| E2
    E2 -->|"To apps A1, A2 14"| E3
    E2 -->|"To apps A1, A2 15"| E4

    D3 -->|"HTTP(S) traffic 16"| D5
    D3 -->|"Health/management 17"| D4
    D5 -->|"Allowed traffic 18"| F2
    F2 -->|"To apps B1, B2 19"| F3
    F2 -->|"To apps B1, B2 20"| F4

    %% Cross‑site / loopback DNS (21–24)
    E3 -->|"May resolve via GTM 21"| C1
    F3 -->|"May resolve via GTM 22"| D1
    C1 -->|"Loopback to local WAF 23"| C5
    D1 -->|"Loopback to local WAF 24"| D5
```

---

## 3. DNS / GTM behavior and client resolver considerations

### 3.1 DNS deployment for four GTM‑based DNS servers

DNS layout:

- **Primary Site**:
  - `IP_C1_VIP` – GTM/DNS Cluster 1 (Primary).  
  - `IP_C2_VIP` – GTM/DNS Spare 1 (Primary).  
- **DR Site**:
  - `IP_D1_VIP` – GTM/DNS Cluster 2 (DR).  
  - `IP_D2_VIP` – GTM/DNS Spare 2 (DR).  

Clients should be configured with these **4 DNS server IPs** (in order of preference, if possible):

- Corporate DNS forwards extranet domain to these GTM‑managed DNS servers (via `NS` or conditional‑forwarding).  
- **Recommended TTL**: 30–60 seconds for DNS records to enable faster site failover.

---

### 3.2 Client resolver behavior (Windows vs Linux)

| Aspect                          | Windows                                                                 |
|---------------------------------|-------------------------------------------------------------------------|
| Configured DNS servers          | Up to 3 DNS servers per interface; prefers first, then fallback.      |
| Query order                     | Tries first DNS; on timeout, retries next; caches “fastest responder” and adjusts order. |
| Negative vs timeout             | Negative answer stops query; only timeout triggers retry.             |
| Corporate intranet DNS usage    | Corporate DNS may forward or recursively resolve; may delegate extranet domain to GTM‑DNS. |

| Aspect                          | Linux                                                                   |
|---------------------------------|-------------------------------------------------------------------------|
| Configured DNS servers          | `/etc/resolv.conf` with up to 3 `nameserver` entries.                  |
| Query order                     | Uses first server; falls to next only on timeout or network error.    |
| Negative vs timeout             | Negative answer stops query; retries only on timeout.                 |
| Corporate intranet DNS usage    | Corporate DNS delegates to GTM‑DNS via `NS` or forward‑zone for extranet domain. |

---

### 3.3 How GTM works with site priority

- GTM maintains a **Wide IP** for the extranet domain (e.g., `extranet.example.com`).  
- Each Wide IP has **pools** pointing to **LTM virtual‑server VIPs** in each site.  
- **Topology** (site‑priority) method:
  - Prefer **local‑site LTM pool** for local‑site clients.  
  - Use **DR‑site LTM pool** when primary is unhealthy.  
  - GTM health‑checks monitor LTM virtual servers and WAF status.

**Failover**:

- When **primary site LTM / WAF / apps** fail, GTM removes that pool from the answer set and replies with **DR‑site VIPs** for new DNS queries.  
- Existing TCP sessions eventually time out; clients must re‑resolve once **DNS TTL** elapses.  
- With **TTL = 30–60 seconds**, clients refresh DNS within ~1 minute.

For apps:

- Prefer **stateless or short‑lived sessions**.  
- For **session‑affinity** patterns, design a **soft re‑start** (e.g., auto‑re‑login or token refresh) instead of expecting long‑lived TCP sessions to survive DNS‑based site‑switch.  

---

## 4. Failover and application behavior

Assumptions:

- **DNS TTL = 60 seconds** (recommended).  
- Apps are **web‑based (HTTP/HTTPS)**.  
- Some apps are **sessionless**; some use **session affinity** (e.g., cookies).

### 4.1 Single‑fault scenarios (per layer)

| Block / component | Layer           | Fault / outage                                      | Impact on data flow / client behavior                                                                 | Client‑side effect (reconnect vs retransmit) |
|-------------------|-----------------|-----------------------------------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------------|
| C1 (GTM DNS clus) | DNS (GTM)       | One GTM node fails; VIP still active in cluster.    | Minimal impact; DNS remains available. No DNS change to clients.                                     | No extra reconnect; existing connections continue. |
| C2 (GTM spare)    | DNS (GTM)       | Independent spare node fails.                       | No impact if cluster is healthy.                                                                     | No impact.                                   |
| C3 (LTM clus)     | Load‑balancer   | One LTM node fails; VIP active in cluster.          | Ongoing connections continue; LTM cluster handles failover.                                          | No reconnect; TCP retransmits handled by LTM. |
| C4 (LTM spare)    | Load‑balancer   | Spare LTM node fails.                               | No impact if cluster is healthy.                                                                     | No impact.                                   |
| C5 (WAF clus)     | WAF             | One WAF node fails.                                 | Cluster remains up; WAF filters continue; may flap during health‑check.                              | Some 5xx errors possible; client retries.    |
| C6 (WAF spare)    | WAF             | Spare WAF node fails.                               | No impact if cluster is healthy.                                                                     | No impact.                                   |
| E1/E2 or F1/F2    | CFW / ELB       | One CFW or ELB node fails.                          | Cloud‑side HA masks failure; app traffic continues.                                                  | No impact if ELB remains active.             |
| E3/E4 or F3/F4    | App server      | One app server fails.                               | LTM/WAF health‑monitor removes it; traffic shifts to other nodes.                                    | No reconnect; HTTP may re‑initiate if session drops. |
| A1/A2             | Client          | Partner client link or endpoint fails.              | Upstream LTM/WAF may see RST/timeout; apps see connection drop.                                      | Client must reconnect.                       |
| B1/B2             | Corp DNS        | Corporate DNS fails.                                | Corporate users cannot resolve extranet domain; direct‑to‑GTM may still work.                        | Corporate users must either use GTM‑direct DNS or have redundant corporate DNS. |

> **Note for application teams**:  
> For each application, fill in:
> - Whether sessions are **sessionless** or **session affinity**.  
> - What the **user experience** is on failover (e.g., “session lost, client must re‑login”).  
> - Any **RTO/RPO** or **state‑recovery** requirements.

### 4.2 Double‑fault / site‑wide scenarios

| Scenario description                                                                 | Blocks affected (example)                         | Impact and behavior                                                                                                                                             |
|--------------------------------------------------------------------------------------|---------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Primary site GTM cluster + spare fail (C1, C2).                                      | C1, C2                                            | External clients using Primary DNS fail to resolve; DR GTM must be reachable (client DNS or site‑priority DNS).                                                |
| Primary site LTM cluster + spare fail (C3, C4).                                      | C3, C4                                            | Inbound traffic to Primary site stops; GTM must redirect DNS queries to DR‑site. New sessions go to DR.                                                       |
| Primary site WAF cluster + spare fail (C5, C6).                                      | C5, C6                                            | If WAF is required, LTM may drop traffic or forward without WAF; client may see 5xx or timeout until DNS‑based failover.                                       |
| Primary site ELB(s) or CFW fail (E1/E2).                                             | E1, E2                                            | Active/active apps in DR continue; DR‑site GTM may already route to DR; Primary app traffic blackholes.                                                       |
| Primary site app servers (E3/E4) all fail.                                           | E3, E4                                            | LTM/WAF health‑checks mark pool down; GTM routes DNS to DR‑site pool. New sessions go to DR; existing sessions time out.                                      |
| Primary site GTM + DR GTM both partially down (e.g., C1, D1 fail).                  | C1, D1                                            | If DNS quorum is lost, DNS may become unavailable; external clients cannot resolve. DNS spares and client DNS diversity are critical.                          |
| Corporate DNS + GTM misconfiguration (delegation broken).                            | B2, C1, D1                                        | Corporate users cannot resolve extranet domain; direct‑to‑GTM DNS may still work, but corporate users are blocked.                                           |

> **Application‑team placeholder**:  
> For each application, define:
> - App type: **sessionless** vs **session affinity**.  
> - Failover behavior: does it require **full re‑login**, or can it **resume** via token/cookie?  
> - Tolerance for **session loss** and **RTO expectations**.

---

## 5. Client‑reconnect vs network retransmit

| Condition                                                            | What happens                                                                                                                             |
|---------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| LTM node fails, cluster remains.                                    | TCP connections may be dropped or re‑established; client may see brief error then reconnect automatically.                               |
| WAF node fails, cluster remains.                                    | LTM re‑selects healthy WAF; client may see 5xx or brief timeout; HTTP client retries.                                                   |
| One of 4 DNS servers times out.                                     | Client moves to next DNS server; no app impact if DNS still resolves.                                                                   |
| DNS TTL expires after site‑failover.                                | Client re‑resolves; gets DR‑site VIPs; opens new TCP/HTTPS connection; client may experience short outage or redirect.                 |
| Session‑affinity (cookie‑based) with active/passive apps.           | After DNS‑induced site‑switch, app may not have the session; client usually needs to re‑authenticate or restart flow.                  |
| Sessionless (stateless API) with short‑TTL DNS.                    | Client re‑connects to new site; app behavior is functionally identical; no long‑term session loss.                                     |

Recommendation:

- Set DNS TTL **≤ 60 seconds** for mission‑critical systems.  
- Design **stateless or token‑based** apps where possible; keep **session‑affinity** for authenticated flows that can tolerate short re‑login.  

---

## 6. Issues that need further detail

Each of these should be taken as a **to‑do / requirement** for the relevant team:

1. **Client DNS‑config best‑practices**:
   - How many DNS servers to push to clients (2 vs 4).  
   - Whether to order DNS IPs by site proximity (site‑local first).  

2. **Corporate DNS delegation**:
   - Conditional‑forwarding vs zone delegation to GTM‑managed DNS.  
   - Split‑DNS design for internal vs extranet resolution.  

3. **Direct‑to‑GTM DNS behavior**:
   - If clients have 4 DNS IPs, they may land on DR‑site DNS; is that acceptable for latency‑sensitive partners?  

4. **Windows‑specific DNS behavior**:
   - How Windows caches and ranks DNS servers (speed‑based ordering); whether to standardize first DNS to **site‑local GTM‑DNS**.  

5. **Linux‑specific DNS timeouts**:
   - Timeouts and retries in `/etc/resolv.conf` (e.g., `timeout`, `attempts`) and how they interact with GTM‑DNS failover.  

6. **GTM‑only scenarios**:
   - Detailed GTM health‑check configuration (LTM vs, WAF, and app‑level health) and how failures propagate to DNS answers.  

7. **Application‑session behavior**:
   - For each application, document whether sessions are **sessionless**, **cookie‑based**, or **token‑based**, and how they behave on DNS‑driven site‑switch.  

---


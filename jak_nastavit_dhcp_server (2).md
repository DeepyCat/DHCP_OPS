# Modul 03: DNS Server – postup

Doména: `<prijmeni>.cyberschool.internal` (příjmení bez diakritiky)
Číslo pracoviště: **N** (ve skriptech se zjistí automaticky z IP adresy na `Ethernet1`)

| Stroj   | Role                   | IP               | DNS            |
|---------|------------------------|------------------|----------------|
| SRV1-DC | AD DS, DNS, DHCP       | `192.168.N.10`   | `192.168.N.10` |
| SRV2-FS | File Server (budoucí)  | `192.168.N.20`   | `192.168.N.10` |
| PC1-WIN | Klient (Windows 11)    | DHCP / statická  | `192.168.N.10` |

> Na rozhraní **Ethernet0 (NAT)** nikdy nesahej. Pracuje se jen s **Ethernet1** (interní síť).
> Všechny příkazy spouštěj v PowerShellu **jako správce**.

---

## Část A – SRV1-DC

### A1. Kontrola role DNS a dopředné zóny

```powershell
Get-Service -Name DNS | Select-Object Name, Status, StartType
Get-DnsServerZone -Name "$((Get-ADDomain).DNSRoot)" | Select-Object ZoneName, ZoneType, ReplicationScope, DynamicUpdate
Get-DnsServerResourceRecord -ZoneName "$((Get-ADDomain).DNSRoot)" -RRType A -Name "srv1-dc"
```

Očekávání: DNS `Running / Automatic`, zóna `Primary`, `Domain`, `Secure`.

### A2. AD-integrovaná reverzní zóna pro 192.168.N.0/24

```powershell
$wsX = (Get-NetIPAddress -InterfaceAlias Ethernet1 -AddressFamily IPv4).IPAddress.Split('.')[2]
Add-DnsServerPrimaryZone -NetworkId "192.168.$wsX.0/24" -ReplicationScope "Domain" -DynamicUpdate "Secure"
```

### A3. Statický záznam A + PTR pro SRV2-FS

```powershell
$domain = (Get-ADDomain).DNSRoot
$wsX = (Get-NetIPAddress -InterfaceAlias Ethernet1 -AddressFamily IPv4).IPAddress.Split('.')[2]
Add-DnsServerResourceRecordA -Name "srv2-fs" -ZoneName $domain -IPv4Address "192.168.$wsX.20" -CreatePtr
```

### A4. Alias CNAME `ftp` → `srv2-fs`

```powershell
$domain = (Get-ADDomain).DNSRoot
Add-DnsServerResourceRecordCName -Name "ftp" -HostNameAlias "srv2-fs.$domain" -ZoneName $domain
```

### A5. Doplnění chybějícího PTR pro SRV1-DC (.10)

Řadič vznikl dřív než reverzní zóna, proto PTR chybí.

```powershell
$domain = (Get-ADDomain).DNSRoot
$wsX = (Get-NetIPAddress -InterfaceAlias Ethernet1 -AddressFamily IPv4).IPAddress.Split('.')[2]
Add-DnsServerResourceRecordPtr -Name "10" -ZoneName "$wsX.168.192.in-addr.arpa" -PtrDomainName "srv1-dc.$domain"
```

### A6. Kontrola na serveru

```powershell
$domain = (Get-ADDomain).DNSRoot
$wsX = (Get-NetIPAddress -InterfaceAlias Ethernet1 -AddressFamily IPv4).IPAddress.Split('.')[2]

# reverzní zóna
Get-DnsServerZone -Name "$wsX.168.192.in-addr.arpa" | Select-Object ZoneName, ZoneType, ReplicationScope, DynamicUpdate

# A záznamy
Get-DnsServerResourceRecord -ZoneName $domain -RRType A | Where-Object { $_.HostName -in 'srv1-dc','srv2-fs' } | Select-Object HostName, RecordType, @{N='IPAddress';E={$_.RecordData.IPv4Address}}

# PTR záznamy
Get-DnsServerResourceRecord -ZoneName "$wsX.168.192.in-addr.arpa" -RRType Ptr | Where-Object { $_.HostName -in '10','20' } | Select-Object HostName, RecordType, @{N='PtrDomainName';E={$_.RecordData.PtrDomainName}}

# CNAME
Get-DnsServerResourceRecord -ZoneName $domain -Name "ftp" | Select-Object HostName, RecordType, @{N='HostNameAlias';E={$_.RecordData.HostNameAlias}}
```

Grafická kontrola: `dnsmgmt.msc` → Forward Lookup Zones (A: srv1-dc, srv2-fs, PC1-WIN, CNAME ftp) a Reverse Lookup Zones (PTR 10 a 20).

---

## Část B – PC1-WIN

### B1. Priorita adaptérů (Ethernet1 = 10, Ethernet0 = 50) a kontrola DNS

```powershell
Set-NetIPInterface -InterfaceAlias "Ethernet1" -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "Ethernet0" -InterfaceMetric 50
Get-NetIPInterface -InterfaceAlias "Ethernet1","Ethernet0" -AddressFamily IPv4 | Select-Object InterfaceAlias, InterfaceMetric
Get-DnsClientServerAddress -InterfaceAlias Ethernet1 -AddressFamily IPv4 | Select-Object InterfaceAlias, ServerAddresses
Get-DnsClient -InterfaceAlias Ethernet1 | Select-Object ConnectionSpecificSuffix, RegisterThisConnectionsAddress
```

Očekávání: DNS = `192.168.N.10`, `RegisterThisConnectionsAddress = True`.

### B2. Dynamická registrace klienta v DNS

```powershell
ipconfig /registerdns
ipconfig /flushdns
```

### B3. Test překladu (nahraď N číslem pracoviště)

```powershell
# dopředný překlad
nslookup srv2-fs
Resolve-DnsName -Name "srv2-fs" | Select-Object Name, Type, IPAddress

# reverzní překlad
nslookup 192.168.N.20
nslookup 192.168.N.10

# alias CNAME
nslookup ftp
Resolve-DnsName -Name "ftp.<prijmeni>.cyberschool.internal" -Server 192.168.N.10 | Select-Object Name, Type, NameHost, IPAddress

# odezva přes jméno
ping srv2-fs
```

Očekávání: žádné `Non-existent domain`; `ftp` → CNAME `srv2-fs...` → `192.168.N.20`. Ping na srv2-fs projde jen tehdy, když SRV2-FS už existuje a má povolený ICMP.

---

## Otisk konfigurace pro odevzdání

**SRV1-DC:**

```powershell
Get-Date
whoami
hostname
(Get-CimInstance Win32_ComputerSystemProduct).UUID
(Get-ADDomain).DNSRoot
(Get-ADDomain).DomainSID.Value
```

**PC1-WIN:**

```powershell
Get-Date
whoami
hostname
(Get-CimInstance Win32_ComputerSystemProduct).UUID
(Get-NetIPAddress -AddressFamily IPv4 | Where-Object IPAddress -like '192.168.*').IPAddress
```

## Nepovinné automatické ověření

**SRV1-DC:**

```
powershell -ExecutionPolicy Bypass -Command "curl.exe --progress-bar -L https://raw.githubusercontent.com/DeepyCat/well_ops/main/skripty/03_check_SRV1-DC.bat -o check.bat; .\check.bat; Remove-Item -Force check.bat -ErrorAction SilentlyContinue"
```

**PC1-WIN:**

```
powershell -ExecutionPolicy Bypass -Command "curl.exe --progress-bar -L https://raw.githubusercontent.com/DeepyCat/well_ops/main/skripty/03_check_PC1-WIN.bat -o check.bat; .\check.bat; Remove-Item -Force check.bat -ErrorAction SilentlyContinue"
```



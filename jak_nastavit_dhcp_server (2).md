# Návod na nastavení DHCP serveru pomocí PowerShellu

Tento dokument obsahuje přehledné příkazy pro nastavení DHCP serveru na Windows Serveru (SRV1) a následné ověření z klientské stanice (PC1).

### Krok 1: Instalace role DHCP

**\[SRV1\]**
```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

### Krok 2: Konfigurace bezpečnostních skupin a restart služby

**\[SRV1\]**
```powershell
Add-DhcpServerSecurityGroup

Restart-Service -Name DHCPServer

Set-ItemProperty -Path 'registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ServerManager\Roles\12' -Name 'ConfigurationState' -Value 2
```

### Krok 3: Autorizace DHCP serveru v Active Directory

**\[SRV1\]**
```powershell
$DnsName = "$env:COMPUTERNAME.$((Get-ADDomain).DNSRoot)"
$DnsName

Add-DhcpServerInDC -DnsName $DnsName -IPAddress "192.168.12.10"
```

### Krok 4: Vytvoření nového DHCP rozsahu (Scope)

**\[SRV1\]**
```powershell
Add-DhcpServerv4Scope -Name "Ucebna" -StartRange "192.168.12.100" -EndRange "192.168.12.200" -SubnetMask "255.255.255.0" -State Active
```

### Krok 5: Nastavení parametrů rozsahu (Router, DNS)

**\[SRV1\]**
```powershell
$DomainName = (Get-ADDomain).DNSRoot
$DomainName

Set-DhcpServerv4OptionValue -ScopeId "192.168.12.0" -Router "192.168.12.1" -DnsServer "192.168.12.10" -DnsDomain $DomainName
```

### Krok 6: Nastavení DHCP klienta na klientské stanici

**\[PC1\]**
```powershell
Set-NetIPInterface -InterfaceAlias "Ethernet1" -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceAlias "Ethernet1" -ResetServerAddresses

ipconfig /release Ethernet1
ipconfig /renew Ethernet1
```

### Krok 7: Zjištění aktivních pronájmů (Leases)

**\[SRV1\]**
```powershell
Get-DhcpServerv4Lease -ScopeId "192.168.12.0" | Select-Object IPAddress, HostName, ClientId, AddressState, LeaseExpiryTime
```

### Krok 8: Ověření funkčnosti z klienta

**\[PC1\]**
```powershell
ipconfig /release Ethernet1
ipconfig /renew Ethernet1

Test-NetConnection -ComputerName "192.168.12.10" -InformationLevel Detailed

Resolve-DnsName -Name "srv1-dc.<prijmeni>.cyberschool.internal"
```
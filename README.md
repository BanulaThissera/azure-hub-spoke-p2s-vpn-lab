# Azure Hub-and-Spoke Networking with P2S VPN & Gateway Transit

An enterprise-grade implementation of a secure Azure Hub-and-Spoke network topology. This project demonstrates centralized remote access management via Azure Virtual Network Gateway (Point-to-Site VPN) and transitive routing across isolated spoke workloads without exposing virtual machines to the public internet.

---

## 📌 Architecture Topology & Specifications

| Component | Resource Identifier | Address Space / Subnet | Role & Details |
| :--- | :--- | :--- | :--- |
| **Resource Group** | `rg-hub-banu-network-prod` | East US Region | Logical containment boundary |
| **Hub VNet** | `vnet-hub-prod` | `10.10.0.0/16` | Central transit network |
| **Gateway Subnet** | `GatewaySubnet` | `10.10.255.0/27` | Hosts Azure Virtual Network Gateway |
| **VPN Gateway** | `vpngw-hub-prod` | Public IP: Standard | Route-based VPN Gateway (P2S termination) |
| **Hub Workload** | `vm-hub-workload` | `10.10.1.4` (Subnet: `default`) | Ubuntu Server 24.04 LTS (Private IP only) |
| **Spoke VNet** | `vnet-spoke-prod` | `10.20.0.0/16` | Workload production network |
| **Spoke Workload** | `vm-spoke-windows` | `10.20.1.4` (Subnet: `snet-workload`) | Windows Server 2022 Datacenter (Private IP only) |
| **P2S Client Pool** | Dynamic Pool | `172.16.201.0/24` | Client IP range for VPN endpoints |
| **Peering Model** | Bi-directional Transit | Hub & Spoke | Gateway Transit enabled |

---

## 🛠️ Step-by-Step Implementation Guide

### 1. Hub Virtual Network & Gateway Deployment
- Provisioned `rg-hub-banu-network-prod` in `East US`.
- Configured `vnet-hub-prod` (`10.10.0.0/16`) with `default` (`10.10.1.0/24`) and dedicated `GatewaySubnet` (`10.10.255.0/27`).
- Deployed Route-based VPN Gateway (`vpngw-hub-prod`) using a Standard Public IP.

### 2. Point-to-Site (P2S) VPN & Certificate Authentication
- Generated cryptographic Root and Client certificates via administrative PowerShell:
```powershell
# Create Root Certificate
$rootCert = New-SelfSignedCertificate `
    -Type Custom `
    -KeySpec Signature `
    -Subject "CN=P2SRootCert" `
    -KeyExportPolicy Exportable `
    -HashAlgorithm sha256 `
    -KeyLength 2048 `
    -CertStoreLocation "Cert:\CurrentUser\My" `
    -KeyUsageProperty Sign `
    -KeyUsage CertSign

# Create Child Client Certificate signed by Root
New-SelfSignedCertificate `
    -Type Custom `
    -DnsName "P2SChildCert" `
    -KeySpec Signature `
    -Subject "CN=P2SChildCert" `
    -KeyExportPolicy Exportable `
    -HashAlgorithm sha256 `
    -KeyLength 2048 `
    -CertStoreLocation "Cert:\CurrentUser\My" `
    -Signer $rootCert `
    -TextExtension @("2.5.29.37={critical}1.3.6.1.5.5.7.3.2")
```

- Exported the root certificate as Base64 and uploaded it to Azure VPN Gateway configuration.
- Uploaded the client certificate to the user device to establish secure P2S authentication.

### 3. Virtual Network Peering & Gateway Transit
- Created a VNet peering between `vnet-hub-prod` and `vnet-spoke-prod`.
- Enabled `Use remote gateways` on the hub-to-spoke connection and `Allow gateway transit` on the hub side.
- Enabled the spoke to accept transitive routing through the hub gateway.

### 4. Workload Deployment & Private Connectivity
- Deployed a private Ubuntu 24.04 VM in the hub default subnet with no public IP.
- Deployed a private Windows Server 2022 VM in the spoke workload subnet with no public IP.
- Verified both workloads are reachable through the VPN tunnel without public exposure.

### 5. Validation & Reachability Testing
- Connected to the Azure VPN from a client machine using the generated client configuration.
- Confirmed successful IP assignment from the P2S client pool (`172.16.201.0/24`).
- Verified remote connectivity from the VPN client to both private workloads across the hub-spoke topology.

---

## 🔐 Practical Design Notes

This topology is representative of an enterprise network architecture where:
- the hub is the central transit and security boundary,
- the spoke remains isolated and private,
- P2S clients access internal resources securely through Azure VPN,
- the Azure Virtual Network Gateway handles remote-access routing without exposing workloads directly.

This pattern is particularly useful for:
- secure admin access to private workloads,
- centralized route and policy control,
- reduced public attack surface,
- hybrid enterprise connectivity.

---

## ✅ Network Flow Summary

1. Remote user authenticates to Azure VPN Gateway using a certificate-based P2S profile.
2. Client receives an IP from the VPN pool (`172.16.201.0/24`).
3. Azure VPN Gateway routes traffic to the Hub VNet (`10.10.0.0/16`).
4. Hub-to-Spoke peering forwards traffic to the Spoke VNet (`10.20.0.0/16`).
5. Private workloads receive traffic without public endpoints or direct internet exposure.

---

## 📎 Useful Azure CLI Commands

```bash
az group create --name rg-hub-banu-network-prod --location eastus
az network vnet create --resource-group rg-hub-banu-network-prod --name vnet-hub-prod --address-prefix 10.10.0.0/16 --subnet-name default --subnet-prefix 10.10.1.0/24
az network vnet subnet create --resource-group rg-hub-banu-network-prod --vnet-name vnet-hub-prod --name GatewaySubnet --address-prefixes 10.10.255.0/27
az network public-ip create --resource-group rg-hub-banu-network-prod --name pip-vpngw-hub-prod --sku Standard --allocation-method Static
az network vnet-gateway create --resource-group rg-hub-banu-network-prod --name vpngw-hub-prod --public-ip-address pip-vpngw-hub-prod --vnet vnet-hub-prod --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --no-wait
```

---

## 📘 Reference Concepts

- Azure Hub-and-Spoke Network Topology
- Virtual Network Peering
- Azure VPN Gateway (P2S)
- Gateway Transit
- Private-only workload connectivity
- Certificate-based remote access

---

## 🧭 Purpose

This lab models a realistic enterprise Azure network pattern in which remote administrators connect through a centralized VPN gateway and access workloads hosted in private subnets behind a resilient network hub. It emphasizes security, segmentation, and operational simplicity while keeping all production workloads private and isolated.

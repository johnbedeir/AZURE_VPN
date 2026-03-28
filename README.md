# Azure VPN + Private VM (Terraform)

<img src=imgs/cover.png>

## Goal

Private Linux web app (Nginx) reachable **only via Point-to-Site (P2S) VPN**.

## What this stack creates

- Resource group, **virtual network**, **GatewaySubnet** (VPN gateway), **private subnet** (VM + **NAT gateway** for outbound package installs)
- **Ubuntu 22.04** VM in the private subnet (**no public IP**), Nginx on port 80 via `custom_data`
- **NSG**: TCP **80** and **22** allowed from the **P2S client pool** and **VNet CIDR** (SSH is not open from the public internet)
- **VPN Gateway** (Route-based) with **OpenVPN** and **certificate** (root CA in gateway config, client cert in `client.ovpn`)
- Demo **CA + client** certificates in Terraform state; writes **`client.ovpn`** locally (gitignored)

Azure P2S is the counterpart of **AWS Client VPN**; the profile format is OpenVPN-compatible.

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) `>= 1.3`
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) logged in (`az login`) and a subscription selected (`az account set --subscription ...`)
- **Subscription ID** in `terraform.tfvars` as `azure_subscription_id`
- [Azure VPN Client](https://www.microsoft.com/store/productId/9NP355QT2SQB) or another **OpenVPN-compatible** client
- An **SSH public key** in `terraform.tfvars` (`ssh_public_key`)

## Cost note

**VPN Gateway (VpnGw1)** is relatively expensive and can take **30–60+ minutes** to create on first `apply`. NAT Gateway also has hourly + processing charges. This is intended as a lab / learning stack.

## Deploy

```bash
cd /path/to/AZURE_VPN
terraform init
terraform apply
```

Show outputs anytime:

```bash
terraform output

terraform output client_vpn_config_path
terraform output -raw client_vpn_config_path

terraform output ec2_private_ip
terraform output vpn_test_url
terraform output client_vpn_dns_name
```

The OpenVPN profile is **`client.ovpn`** in this directory (see `client_vpn_config_path`). It contains a **private key**; do not commit it.

## Connect

1. After apply finishes, open **`client.ovpn`** (or the path from `terraform output -raw client_vpn_config_path`).
2. Import it into **Azure VPN Client** or OpenVPN (**Add profile** / import `.ovpn`).
3. Connect.
4. In a browser, open **`http://<private-ip>`** from `terraform output vpn_test_url`. Use **`http://`**, not `https://`.

If the browser spins, see the AWS sibling project README **Troubleshooting** (same ideas: **local LAN overlapping `10.0.0.0/16`**, re-apply and **reload the profile**, check routes with `route -n get <ip>` on macOS).

## Test

- **Without VPN** — the VM has no public IP; you should not reach its private IP from the internet.
- **With VPN** — the default Nginx welcome page should load on the VM’s private IP.

## Destroy

```bash
terraform destroy
```

Gateway deletion can take a long time; keep the session open.

## Variables

Edit **`terraform.tfvars`** (see **`variables.tf`** for descriptions). Use a **non-overlapping** `vnet_cidr` / subnets if your home network uses the same ranges (for example **`172.16.0.0/16`** with matching gateway/private subnets).

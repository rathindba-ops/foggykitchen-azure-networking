# foggykitchen-azure-networking

Azure hub-and-spoke landing zone (OpenTofu), built from the FoggyKitchen
`azure-networking-hub-spoke-basic` pattern.

## Provenance

| Item | Source |
|---|---|
| Payload, root wiring, diagram | `foggykitchen/foggykitchen-landing-zone-orchestrator` `examples/azure/networking/hub_spoke/basic` @ `ee90effa8a9de86ec91822dbb390d5c891ed481f` |
| Pattern (`patterns/azure/hub_spoke`) | same repo and commit, vendored |
| Module refs | upstream used `?ref=main`; pinned here to release tags: vnet v0.1.2, vnet-peering v0.1.1, routing v0.4.3, nsg v1.0.1, public-ip v1.0.1, natgw v1.1.1, bastion v1.0.1, private-dns v1.0.1 (loadbalancer v1.2.0 and compute v0.3.5 unchanged from upstream; disabled in this payload) |

## What gets deployed

- Resource group `rg-fk-azure-lz-dev` in `swedencentral`
- Hub VNet `10.10.0.0/16`: `AzureBastionSubnet` `10.10.0.0/26`, shared services `10.10.1.0/24`
- App spoke `10.20.0.0/16`: frontend `10.20.1.0/24`, backend `10.20.2.0/24`
- Data spoke `10.30.0.0/16`: database `10.30.1.0/24`, private endpoints `10.30.2.0/24`
- Hub-to-spoke peering (both directions)
- Subnet NSGs with deny-Internet-inbound; SSH allowed from the Bastion subnet to app backend and data database
- NAT Gateways (one per spoke) on `app.backend` and `data.database`
- Azure Bastion (Standard)

Edit `landing-zone.yaml` to change region, names or CIDRs.

## Traffic flows

| ID | Source | Destination | Path | Result |
|---|---|---|---|---|
| F1 | app or data spoke | hub | direct peering | allowed |
| F2 | app spoke | data spoke | none: peering is not transitive and routing is disabled | **not reachable** |
| F3 | `app.backend`, `data.database` | Internet | NAT Gateway | allowed |
| F4 | other subnets | Internet | no NAT attached | do not rely on it |
| F5 | Internet | any subnet | NSG deny | blocked |
| F6 | operator | VMs in spokes | Azure Bastion -> peering | allowed |

For spoke-to-spoke traffic, move to the `hub_spoke/routing` or `firewall_transit` pattern.

## Cost

Billable while deployed: Azure Bastion Standard (largest item), two NAT Gateways
and their Standard Public IPs, plus Bastion's Public IP. Destroy when finished.

## Usage

```bash
az login
export ARM_SUBSCRIPTION_ID=<subscription-id>

tofu init
tofu validate
tofu plan -out tfplan
tofu apply tfplan     # creates billable resources
tofu destroy          # when finished
```

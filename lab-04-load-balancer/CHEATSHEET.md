# Lab 04 cheat sheet: steps and rules

For each Azure tool used in Lab 4: the exact steps, then the rules to remember for AZ-104.

## The whole lab in order

1. Create the resource group `rg-lab4-lb` in France Central.
2. Create the virtual network `vnet-web` (10.2.0.0/16) with the subnet `snet-web` (10.2.1.0/24).
3. Create the NSG `nsg-web`, add the rule Allow-HTTP (TCP 80), and associate it to `snet-web`.
4. Create `vm-web1` and `vm-web2` with PowerShell in Cloud Shell: no public IP, custom data builds the web page.
5. Create the load balancer `lb-web`: frontend IP, backend pool, health probe, rule.
6. Test: browser, 10 requests, stop vm-web1, then IP flow verify on ports 22 and 80.
7. Delete the resource group.

## Load balancer in one picture: door, workers, doctor, road

| Part | One word | Its only job | Without it |
|---|---|---|---|
| Frontend IP | The door | Visitors come in here (the VMs have no public IP) | Nobody can reach the website |
| Backend pool | The workers | The servers that answer visitors | The load balancer has nobody to send traffic to |
| Health probe | The doctor | Checks every 5 s which server is healthy | Traffic keeps going to a dead server, so some users get errors |
| Rule | The road | Links door port 80 → workers port 80, using the doctor | The door exists, but nothing moves |

> A visitor comes in the **door**, takes the **road**, and goes to a **worker** that the **doctor** says is healthy.

## Resource group

A folder that holds related Azure resources, so you manage and delete them together.

**Steps:** Resource groups → **+ Create** → name `rg-lab4-lb`, region France Central → **Create**.

**Rules**
- One resource belongs to exactly one resource group.
- Resources inside can be in other regions; the group's region only stores its metadata.
- Deleting the group deletes everything in it.
- RBAC roles, policies, locks and tags set on a group apply to the resources inside.

## Virtual network and subnet

A VNet is your private network in Azure. A subnet is a smaller address range inside it.

**Steps**
1. Virtual networks → **+ Create**: `vnet-web`, France Central.
2. IP addresses: address space `10.2.0.0/16`; delete **default**; add `snet-web`, starting address `10.2.1.0`, size **/24**.
3. Keep **Enable private subnet** checked → **Create**.

**Rules**
- A subnet must fit inside the VNet's address space.
- Azure reserves **5 addresses** in every subnet.
- `/16` ≈ 65,536 addresses, `/24` = 256.
- A VNet stays in one region and one subscription.
- Peered VNets must not have overlapping address spaces.
- Private subnet: no default outbound internet; inbound traffic through a load balancer still works.

## Network security group (NSG)

A list of allow and deny rules that filters traffic to a subnet or a network card.

**Steps**
1. Network security groups → **+ Create**: `nsg-web`, France Central.
2. Inbound security rules → **+ Add**: Source Any, Service **HTTP** (TCP 80), Action **Allow**, Priority 100, Name `Allow-HTTP`.
3. Subnets → **+ Associate** → `vnet-web` / `snet-web`. Check *Associated with: 1 subnets*.

**Rules**
- Lowest priority number is checked first (100–4096); the **first match wins**.
- Default rules: AllowVnetInBound (65000), AllowAzureLoadBalancerInBound (65001), DenyAllInBound (65500).
- An NSG does nothing until it is **associated**.
- Subnet NSG + NIC NSG: traffic must be allowed by **both** (inbound: subnet first, then NIC).
- HTTP = TCP 80 · HTTPS = TCP 443 · SSH = TCP 22 · RDP = TCP 3389.

## Virtual machines and custom data

A VM is a server in Azure. Custom data is a script the VM runs by itself at first boot.

**Steps (portal)**: Virtual machines → **+ Create** → Ubuntu 24.04, allowed size, password, public inbound ports **None** → Disks: Standard SSD → Networking: `snet-web`, public IP **None**, NIC NSG **None** → Advanced: paste **Custom data** → **Create**. Repeat for `vm-web2`.

**Rules**
- Custom data runs **only at the first boot**.
- No public IP = nobody on the internet reaches the VM directly.
- Availability zone = a separate datacenter in the same region.
- A size can be blocked by **location** or **zone** restrictions: check with `Get-AzComputeResourceSku`.
- Student subscriptions cannot use availability zones.

## Cloud Shell and PowerShell

Cloud Shell is a terminal inside the portal. PowerShell (Az module) creates resources with commands.

**Steps:** click `>_` → **PowerShell** (prompt `PS /home/...>`; in Bash click **Switch to PowerShell**) → **No storage account required** → paste the script.

| Command | What it does |
|---|---|
| `Get-AzVirtualNetwork` | Finds an existing VNet |
| `New-AzNetworkInterface` | Creates a network card |
| `New-AzVMConfig` → `New-AzVM` | Builds the VM settings, then creates the VM |
| `Get-AzComputeResourceSku` | Shows which sizes are blocked, and why |
| `Get-AzVM -Status` | Shows running / deallocated VMs |
| `Stop-AzVM` / `Start-AzVM` | Stops (deallocates) or starts a VM |
| `Remove-AzResourceGroup` | Deletes a resource group and everything in it |

**Rules**
- Bash uses **Azure CLI** (`az ...`); PowerShell uses the **Az module** (`New-AzVM`).
- Variables disappear when you close Cloud Shell.
- `try { } catch { }` keeps a script running after an error; always print the error message.

## Load balancer

Spreads incoming traffic across several servers and skips any server that stops answering.

**Steps**
1. Load balancers → **+ Create**: `lb-web`, SKU **Standard**, Type **Public**, Tier **Regional**.
2. Frontend IP: `fe-web`, new public IP `pip-lb-web` (No zone on a student subscription).
3. Backend pool: `bp-web`, VNet `vnet-web`, configuration **NIC**, add both VMs.
4. Inbound rule: `rule-http`, TCP 80 → 80, new probe `probe-http` (HTTP, 80, `/`, every 5 s), session persistence **None**.
5. **Create**, then open `http://<frontend IP>`.

**Rules**
- Default distribution is a **5-tuple hash**, not round-robin. **Session persistence** (client IP) keeps a visitor on one server.
- Load Balancer = **layer 4** (IP + port). Need URL routing, HTTPS certificates or a web firewall? Use **Application Gateway** (layer 7).
- The default NSG rule **AllowAzureLoadBalancerInBound** lets the probe reach the VMs.
- A Standard load balancer is closed by default: traffic passes only if an NSG allows it.

## Network Watcher: IP flow verify

Tells you if a packet is allowed or denied, and which NSG rule decided, without sending real traffic.

**Steps:** search **IP flow verify** → choose the VM (must be running) and NIC → Protocol **TCP**, Inbound, local port 22, remote IP 8.8.8.8, remote port 12345 → **Verify IP flow** → scroll to **Results**. Repeat with port 80.

**Rules**
- The result shows Access allowed/denied + the rule name + the NSG.
- SSH and HTTP are TCP; DNS is UDP; ping is ICMP.
- Other tools: Next hop, Connection troubleshoot, Packet capture, Effective security rules.
- `timed out` = probably dropped by an NSG/firewall; `connection refused` = reached the VM, nothing listens on that port.

## Cleanup and VM power states

**Steps:** `Remove-AzResourceGroup -Name "rg-lab4-lb" -Force`, then `Get-AzVM -Status` (empty = nothing left).

| Power state | How it happens | Compute cost |
|---|---|---|
| VM running | Started | Yes |
| VM stopped | Shut down from inside the OS | **Yes, still charged** |
| VM deallocated | Stop in the portal or `Stop-AzVM` | No (disks still charged) |

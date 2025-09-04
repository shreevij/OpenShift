**How We Simplified Windows VM Migration to OpenShift with PowerShell Automation**

Migrating Windows virtual machines (VMs) into OpenShift can be tricky — especially when it comes to networking. One of the biggest headaches is that the network interface changes during migration, meaning the original IP addresses and network settings get lost.

Manually fixing this via the OpenShift console is slow and frustrating, especially if you’re moving lots of VMs. So, we built a PowerShell automation to handle the network settings automatically — making the whole migration smoother, faster, and less error-prone.

**The Problem: Why Does Network Configuration Get Lost?**

When you move a Windows VM into OpenShift, the platform dynamically assigns new network interfaces. This means the VM’s old IP address, subnet mask, gateway, and DNS settings aren’t preserved.

That leaves engineers with the task of manually assigning IPs through OpenShift’s console — which is slow and clunky. Imagine having to do this dozens or hundreds of times during a big migration! It wastes time and invites mistakes.

**Our Simple, Effective Solution: PowerShell Scripts to the Rescue**

We created two PowerShell scripts that do the heavy lifting for us:

Script 1 (Before Migration): This grabs the VM’s current network info (IP, gateway, DNS, etc.) and saves it as a JSON file right on the VM.

Script 2 (After Migration): This runs automatically when the VM starts in OpenShift. It reads that saved info and reapplies the exact same network settings to the new interface.

With this, the VM “remembers” its network settings and configures itself automatically — no more slow manual work!

**Here’s How It Works — Step by Step**
Script 1: Save Your Network Settings Before Migration

This script finds the main network adapter, grabs all the important settings, and saves them in a simple JSON file on the VM.

                # Find the main active network adapter with IPv4 address
                $adapter = Get-NetIPConfiguration | Where-Object { $_.IPv4Address -and $_.NetAdapter.Status -eq 'Up' } | Select-Object -First 1
                
                if ($adapter) {
                    $config = [PSCustomObject]@{
                        InterfaceAlias = $adapter.InterfaceAlias
                        IPAddress      = $adapter.IPv4Address.IPAddress
                        PrefixLength   = $adapter.IPv4Address.PrefixLength
                        DefaultGateway = $adapter.IPv4DefaultGateway.NextHop
                        DNSServers     = ($adapter.DNSServer.ServerAddresses -join ",")
                    }
                
                    # Save configuration to JSON file
                    $config | ConvertTo-Json | Set-Content -Path "C:\scripts\NICConfig.json"
                    Write-Host "Network configuration saved to C:\scripts\NICConfig.json"
                } else {
                    Write-Host "No active network adapter found."
                }
                # Register a scheduled task to apply these settings automatically after migration (assuming task XML is ready)
                Register-ScheduledTask -TaskName "SetStaticIP" -Xml (Get-Content "c:\scripts\SetStaticIP.xml" | Out-String) -Force
_____________________________________________________________________
**Script 2: Automatically Apply Your Network Settings After Migration**
_____________________________________________________________________
When your VM boots in OpenShift, this script kicks in. It reads the saved settings and configures the new network interface accordingly.

                # Load saved network configuration
                $config = Get-Content -Raw -Path "C:\scripts\NICConfig.json" | ConvertFrom-Json
                
                # Find the active network adapter
                $nic = Get-NetAdapter | Where-Object { $_.Status -eq 'Up' } | Select-Object -First 1
                
                if ($nic) {
                    # Remove existing IP and DNS settings to avoid conflicts
                    Get-NetIPAddress -InterfaceAlias $nic.Name -AddressFamily IPv4 | Remove-NetRoute -Confirm:$false -ErrorAction SilentlyContinue
                    Get-NetIPAddress -InterfaceAlias $nic.Name -AddressFamily IPv4 | Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue
                    Set-DnsClientServerAddress -InterfaceAlias $nic.Name -ResetServerAddresses
                
                    # Apply saved IP, subnet prefix, gateway, and DNS servers
                    New-NetIPAddress -InterfaceAlias $nic.Name -IPAddress $config.IPAddress -PrefixLength $config.PrefixLength -DefaultGateway $config.DefaultGateway
                    Set-DnsClientServerAddress -InterfaceAlias $nic.Name -ServerAddresses ($config.DNSServers -split ",")
                
                    Write-Host "Network configuration restored on $($nic.Name)"
                } else {
                    Write-Host "No active network adapter found."
                }
                
                # Remove the scheduled task so it doesn’t run again unnecessarily
                Unregister-ScheduledTask -TaskName SetStaticIP -Confirm:$false

**How to Set This Up**

Create a folder on your VM, like C:\scripts, to store the scripts and config file.

Save Script 1 and Script 2 as separate .ps1 files in that folder.

Create a scheduled task XML file (SetStaticIP.xml) that triggers Script 2 to run automatically at startup. This ensures the VM configures itself right after booting in OpenShift.

Run Script 1 before migrating the VM — it saves your network settings and sets up the scheduled task.

Migrate your VM to OpenShift.

When the VM boots up in OpenShift, Script 2 will automatically restore the original IP configuration.

**Why This Matters**

  **Saves tons of time** compared to manually fixing IPs one-by-one.

  **Avoids human errors** that can cause network issues.

  **Scales easily** for large migration projects.

  **Keeps your VMs connected** with minimal downtime.

**Final Thoughts**

Moving Windows VMs to OpenShift no longer has to be a headache over lost IP settings. With these simple PowerShell scripts, the process becomes smoother and faster — so your team can focus on more important work.

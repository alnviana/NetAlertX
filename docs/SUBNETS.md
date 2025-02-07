# Subnets configuration

You need to specify the network interface and the network mask. You can also configure multiple subnets and specify VLANS (see exceptions below).

> [!TIP]
> You may need to increase the time between scans `ARPSCAN_RUN_SCHD` and the timeout `ARPSCAN_RUN_TIMEOUT` (and similar setting on related plugins) when adding more subnets. If the timeout setting is exceeded, the scan is cancelled to prevent application hanging from rogue plugins. Check [debugging plugins](/docs/DEBUG_PLUGINS.md) for more tips.

## Examples

> [!NOTE] 
> Please use the UI to configure settings as that ensures that the config file is in the correct format. Edit `app.conf` directly only when really necessary.
> ![settings](/docs/img/SUBNETS/subnets-setting-location.png)

* Examples for one and two subnets  (❗ Note the `['...','...']` format):
   * One subnet:   `SCAN_SUBNETS = ['192.168.1.0/24 --interface=eth0']`
   * Two subnets:  `SCAN_SUBNETS = ['192.168.1.0/24 --interface=eth0','192.168.1.0/24 --interface=eth1 -vlan=107']` 

If you get timeout messages, decrease the network mask (e.g.: from a `/16` to `/24`) or increase the `TIMEOUT` setting (e.g.: `ARPSCAN_RUN_TIMEOUT` to `300` (a timeout of 5min)) for the plugin and the interval between scans (e.g.: `ARPSCAN_RUN_SCHD` to `*/10 * * * *` (scans every 10 min)). 

## Explanation

### Network mask

**Example value: `192.168.1.0/24`**

The arp-scan time itself depends on the number of IP addresses to check. 

> The number of IPs to check depends on the [network mask](https://www.calculator.net/ip-subnet-calculator.html) you set on the `SCAN_SUBNETS` setting. 
> For example, a `/24` mask results in 256 IPs to check, whereas a `/16` mask checks around 65,536. Every IP takes a couple of seconds. This means that with an incorrect configuration, the arp-scan will take hours to complete instead of seconds.

Specify the network filter (which **significantly** speeds up the scan process). For example, the filter `192.168.1.0/24` covers IP ranges `192.168.1.0` to `192.168.1.255`.

### Network interface (adapter)

**Example value: `--interface=eth0`**

The adapter will probably be `eth0` or `eth1`. (Check `System info` > `Network Hardware` or run `iwconfig` in the container to find your interface name(s)) 

![Network hardware](/docs/img/SUBNETS/system_info-network_hardware.png)

> [!TIP]
> Alterantive to `iwconfig` run `ip -o link show | awk -F': ' '!/lo|vir|docker/ {print $2}'` in your container to find your interface name(s) (e.g.: `eth0`, `eth1`).

### VLANs

**Example value: `-vlan=107`**

- Append e.g.: ` -vlan=107` to the interface field (e.g.: `eth0 -vlan=107`) for multiple vlans. More details in this [comment in this issue](https://github.com/jokob-sk/NetAlertX/issues/170#issuecomment-1419902988)


#### VLANs on a Hyper-V setup

> Community sourced content by [mscreations](https://github.com/mscreations) from this [discussion](https://github.com/jokob-sk/NetAlertX/discussions/404).

> [!NOTE] 
> The setup this was tested on: Bare Metal -> Hyper-V on Win Server 2019 -> Ubuntu 22.04 VM -> Docker -> NetAlertX. 

**Approach 1 (may cause issues):**

Configure multiple network adapters in Hyper-V with distinct VLANs connected to each one using Hyper-V's network setup. However, this action can potentially lead to the Docker host's inability to handle network traffic correctly. The issue may stem from the creation of routes for network time servers or domain controllers on every interface, thereby preventing proper synchronization of the underlying Ubuntu VM. This interference can affect the performance of other applications such as Authentik.

**Approach 2 (working example)**

Network connections to switches are configured as trunk and allow all VLANs access to the server. 

By default Hyper-V only allows untagged packets through to the VM interface and no VLAN tagged packets get through. In order to fix this follow these steps:

1) Run the following command in Powershell on the Hyper-V machine: 

```shell
Set-VMNetworkAdapterVlan -VMName <Docker VM Name> -Trunk -NativeVlanId 0 -AllowedVlanIdList "<comma separated list of vlans>"
```

(There might be other ways how adjust this.)

2) Within the VM, set up sub-interfaces for each of the VLANs so they can be scanned. On Ubuntu 22.04 Netplan can be used.

In /etc/netplan/00-installer-config.yaml, add vlan definitions:

```
network:
  ethernets:
    eth0:
      dhcp4: yes
  vlans:
    eth0.2:
      id: 2
      link: eth0
      addresses: [ "192.168.2.2/24" ]
      routes:
        - to: 192.168.2.0/24
          via: 192.168.1.1 
```

3) Run `sudo netplan apply` and the interfaces are then available to scan in NetAlertX. 
4) In this case, use `192.168.2.0/24 --interface=eth0.2` in NetAlertX

#### VLAN 🔍Example:

![Vlan configuration example](/docs/img/SUBNETS/subnets_vlan.png)

#### Support for VLANS (& exceptions)

Please note the accessibility of the macvlans when they are configured on the same computer. My understanding this is a general networking behavior, but feel free to clarify via a PR/issue.

- NetAlertX does not detect the macvlan container when it is running on the same computer.
- NetAlertX recognizes the macvlan container when it is running on a different computer.


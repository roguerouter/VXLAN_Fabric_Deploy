# VXLAN_Fabric_Deploy

The VXLAN fabric deployment playbooks assist in helping engineers rapidly deploy a VXLAN Fabric using CML or physical switches  The playbook currently deploys two spine and leaf switches that are fully operational after running the playbook.  After running the initialize_vxlan_fabric, VNIs can be added to test VXLAN L2 connectivity between devices.  Additional spines and leafs can be added but modifications need to be made to templates (see limitations)

## Limitations:

- No support for ISIS, yet.
- Code is fairly rigid
  -  Expanding to more nodes today requires editing the role playbooks for spine and leaf with your additional interfaces.

## Setup:

At a minimum, the playbooks require at you have SSH connectivity to your devices.  To run the playbooks **as-is** your fabric should look similar to the image below.  If you are using CML, an external connector needs to be added and configured for **"bridge"** mode.  An unmanaged switch (or managed if you desire) should be used to connect the external connector and management interfaces of the spine and leaf switches.  Finally make the following connections between switches:

- Spine 1 (Eth1/1 and Eth1/2) to Leaf 1 (Eth 1/1) and Leaf 2 (Eth 1/1)
- Spine 2 (Eth1/1 and Eth1/2) to Leaf 1 (Eth 1/2) and Leaf 2 (Eth 1/2)

![VXLAN Fabric Example](simple_vxlan_fabric.png)

Use IP addresses that you are able to assign to CML or Switches and are accessible from your Ansible server.  Assign the addresses to the management interfaces of each device.  Document what IP you assign to each device as these will be needed in the inventory.yml file to identify the correct switch.

```
vrf context management
  ip route 0.0.0.0/0 X.X.X.X
interface mgmt0
  vrf member management
  ip address X.X.X.X/YY
```

Validate you can reach each device via SSH before proceeding.

## Directions

Playbooks should be run in this order:

- enable_nxapi.yml
  - Enables the NXAPI and Restconf features to allow deployment of code, you'll only need to run this once.
- generate_day1_config.yml
  - Generate Day 1 config will create a checkpoint of the device as it exists before deploying the code.  You should ensure that everything you want set for initial startup is complete (IP address, username/password, etc.).  This will allow you to revert to a clean state if the code fails or if you want to start from scratch.
- get_nxapi_token.yml (run as follows: ansible-playbook get_nxapi_token.yml -i inventory/cml_inventory.yml -u admin -k --limit="n9kv-s1")
  - Retrieves a token based on the provided username and password limited to a single node.  The resulting output is displayed in the "token.json.imdata[0].aaaLogin.attributes.token" Token Data output.  You'll need to copy the token data and place it in to the group_vars/all.yml file in the cookie variable.  After running this command, run "echo -n user:password | base64" (change your username / password) to get your Authorization string to add to the all.yml file.  The authorization value should be "Basic <base64_command_result>", ie "Basic ZGlkeW91cmVhbGx5OnRoaW5rdGhpc3dhc2FwYXNzd29yZA==" (That is not a real username and password, protect your base64 value, the data can be easily decoded by anyone with a base64 decoder).
- hardware_tcam_settings.yml
  - Only necessary when deploying on CML instances, on physical HW you can skip this.  Adjusts TCAM space to enable ARP supression capability.  ARP suppression functionality requires that ARP-ETHER TCAM has space added to it.  For this example we zero out vpc-convergence and add the space to arp-ether.  The box will then reboot to apply changes.
- initialize_vxlan_fabric.yml
  - This is the main code, this runs code in the roles/common, roles/spine, and roles/leaf directories.
- add_vni.yml
  - Adds a simple L2VNI and configures an interface for testing.

## Verification

Basic verification that everything is operating will be a ping between devices is successful.  Additional verification steps are as follows:

- Verify underlay routing:
  - show ip route
  - *From leaf 1 ping leaf 2*, show ip route 10.168.0.12 ,*Verify two paths exist through Eth1/1 and Eth1/2*
  - *From leaf 2 ping leaf 2*,  show ip route 10.168.0.11 ,*Verify two paths exist through Eth1/1 and Eth1/2*
  - *Verify leaf 1 can ping leaf 2*, ping 10.168.0.12 source-interface loopback2
  - *Verify leaf 2 can ping leaf 1*, ping 10.168.0.11 source-interface loopback2
- Verify overlay routing:
  - show ip bgp summary
- Verify VXLAN operation:
  - show vxlan
  - show nve peers ,*Only works when traffic has been generated from hosts associated with VLAN/VXLAN*
  - sho bgp l2vpn evpn
  - sho bgp l2vpn evpn vni-id 100500
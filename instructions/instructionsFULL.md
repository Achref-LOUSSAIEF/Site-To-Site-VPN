# Highly-Available Dynamic Site-to-Site VPN with AWS

## Stage 1A - AWS and On-Premises Environment Setup

1. Deploy `HA-VPN-AWS.yaml` in `us-east-1` (AWS) and check capabilities if prompted.
2. Deploy `HA-VPN-ONPREM.yaml` in `us-east-1` (ONPREM) and check capabilities if prompted.
3. Wait for both stacks to reach `CREATE_COMPLETE` status (~5-10 mins).

## Stage 1B - Creating Customer Gateway Objects

1. Open the AWS **VPC Console** and **CloudFormation Console**.
2. In CloudFormation, select **ON-PREM stack** → Outputs.
3. Note the IPs for `Router1Public` and `Router2Public`.
4. In VPC Console, navigate to **Customer Gateways**:
   - Create **ONPREM-ROUTER1** with:
     - Routing: `Dynamic`
     - BGP ASN: `65016`
     - IP Address: `Router1Public`
   - Create **ONPREM-ROUTER2** with:
     - Routing: `Dynamic`
     - BGP ASN: `65016`
     - IP Address: `Router2Public`

## Stage 2A - Creating VPN Attachments for Transit Gateway

1. Go to **Transit Gateway Attachments**.
2. Create VPN attachments for `A4LTGW`:
   - Select `VPN` as attachment type.
   - Choose `Existing` for Customer Gateway and select `ONPREM-ROUTER1`.
   - Enable acceleration.
   - Repeat for `ONPREM-ROUTER2`.
3. In **Site-to-Site VPN Connections**, download and rename configurations:
   - Match `Customer Gateway Address` with `Router1Public`, download as `CONNECTION1CONFIG.TXT`.
   - Match `Router2Public`, download as `CONNECTION2CONFIG.TXT`.

## Stage 2B - Populating Configuration Template

- Fill in **DemoValueTemplate.md** with necessary details from downloaded configurations.

## Stage 3A - Configuring IPSec Tunnels for Router 1

1. Wait until VPN attachments are `available` (~15 mins).
2. Access `ONPREM-ROUTER1` via AWS EC2 **Session Manager**.
3. Modify configuration files:
   - Update `ipsec.conf`, `ipsec.secrets`, and `ipsec-vti.sh` with values from `DemoValueTemplate.md`.
   - Apply changes:
     ```sh
     cp ipsec.conf /etc
     cp ipsec.secrets /etc
     cp ipsec-vti.sh /etc
     chmod +x /etc/ipsec-vti.sh
     systemctl restart strongswan
     ```

## Stage 3B - Configuring IPSec Tunnels for Router 2

1. Repeat **Stage 3A** steps for `ONPREM-ROUTER2` using `CONNECTION2CONFIG.TXT`.

## **Stage 4A - Configure BGP Routing for ONPREMISES-ROUTER1 and Test**  

### **1. Connect to ONPREM-ROUTER1**  
1. Open the **EC2 Console**:  
   [AWS EC2 Console](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:sort=instanceState)  
2. Click **Instances** on the left menu.  
3. Locate and select **ONPREM-ROUTER1**.  
4. Right-click and select **Connect**.  
5. Choose **Session Manager** and click **Connect**.  

### **2. Install and Configure BGP**  
Run the following commands to install and configure BGP on `ONPREM-ROUTER1`:  

```bash
chmod +x ffrouting-install.sh
./ffrouting-install.sh
```

#### **Configure BGP in vtysh**  
```bash
vtysh
conf t
frr defaults traditional
router bgp 65016
  neighbor CONN1_TUNNEL1_AWS_BGP_IP remote-as 64512
  neighbor CONN1_TUNNEL2_AWS_BGP_IP remote-as 64512
  no bgp ebgp-requires-policy
  address-family ipv4 unicast
    redistribute connected
  exit-address-family
exit
exit
wr
exit
```

### **3. Reboot the Router**  
```bash
sudo reboot
```

---  

### **4. Verify Routes**  
#### **Via AWS Console UI**  
- Navigate to **EC2 Console** → **Instances** → **ONPREM-ROUTER1**.  
- Check the route table in the AWS UI.

#### **Via CLI (vtysh)**  
```bash
vtysh
show ip route
exit
```

---  

### **5. Test Connectivity**  

#### **From ONPREM-SERVER1 to EC2-A**  
1. Open the **EC2 Console** → **Instances**.  
2. Select **ONPREM-SERVER1** → **Right-click** → **Connect**.  
3. Choose **Session Manager** → Click **Connect**.  
4. Run the following command:  
   ```bash
   ping IP_ADDRESS_OF_EC2-A
   ```

#### **From EC2-A to ONPREM-SERVER1**  
1. Open the **EC2 Console** → **Instances**.  
2. Select **EC2-A** → **Right-click** → **Connect**.  
3. Choose **Session Manager** → Click **Connect**.  
4. Run the following command:  
   ```bash
   ping IP_ADDRESS_OF_ONPREM-SERVER1
   ```




# STAGE 4B - CONFIGURE BGP ROUTING FOR ONPREMISES-ROUTER2 AND TEST

1. Repeat **Stage 3A** steps for `ONPREM-ROUTER2` using `CONNECTION2CONFIG.TXT`.

## STAGE 5 - CLEANUP

- Delete the on-premises stack
- Delete the VPN COnnections
- Delete the customer Gateways
- WAIT FOR THE CONNECTIONS TO BE REMOVED
- Delete the ONPREM STACK
- Delete the AWS Stack
# Advanced Highly-Available Dynamic Site-to-Site VPN


## STAGE1 - AWS and ONPREM Setup
### 1. INITIAL SETUP OF AWS ENVIRONMENT AND SIMULATED ON-PREMISES ENVIRONMENT
    * Open 1 deployment setup: [CLICK THIS](https://learn-cantrill-labs.s3.amazonaws.com/aws-hybrid-bgpvpn/BGPVPNINFRA.yaml)
    * When stack is create completed, go to `Outputs` tab, note down IP value for the `Router1Public` and `Router2Public`
    Mine is:
        ```
        Router1Public: 34.232.97.254
        Router2Public: 18.214.111.154


### 2. CREATE CUSTOMER GATEWAY OBJECTS
* For ON-PREM ROUTER 1
    * Open VPC console 
    ![alt text](image.png)
    * On the humburger menu, Under `Virtual private network (VPN)`, Select `Customer Gateways` 
    * Click Create `Customer gateway`
    * Set Name to `ONPREM-ROUTER1`
    * Set BGP ASN to `65016`
    * Set IP Address to `Router1Public IP`
    * Click `Create Customer gateway`

![alt text](image-1.png)

* For ON-PREM ROUTER 2
    *  Click Create `Customer gateway`
    * Set Name to `ONPREM-ROUTER2`
    * Set BGP ASN to `65016`
    * Set IP Address to `Router2Public IP`
    * Click `Create Customer gateway`

![alt text](image-2.png)


### 3. CONFIRM NO CONNECTIVITY
    * Move to EC2 Console
    ![alt text](image-3.png)
    * Click `Instances` on the hamburger menu
    * Locate and select `ONPREM-SERVER2`
    * Right Click, click `Connect`
    ![alt text](image-4.png)
    * Select `Session Manager`
    * Click `Connect`
    * Open Instance `AWS-EC2-B`, copy its `Private IPv4 addresses`


    * run ping `IPv4_ADDRESS_OF_AWS_EC2-B`
    * It doesn't work ... because there's no connectivity.


## STAGE2 - TRANSIT GATEWAY VPN ATTACHMENTS
### 1. CREATE VPN ATTACHMENTS FOR TRANSIT GATEWAY
* Move back to VPC console, Under `Transit gateways`, click `Transit gateway attachments`
* For ONPREM-ROUTER1
    * Click `Create Transit Gateway Attachment`
    * Click `Transit Gateway ID` dropdown and select `A4LTGW`
    ![alt text](image-5.png)
    * Select `VPN` for attachment type
    * Select `Existing` for Customer gateway
    * Click `Customer gateway ID` dropdown and select `ONPREM-ROUTER1`
    * Click `Dynamic (requires BGP)` for Routing options
    * Click `Enable Acceleration`
    * Click `Create transit gateway attachment`
![alt text](image-6.png)

* For ONPREM-ROUTER2
    * Click `Create Transit Gateway Attachment`
    * Click `Transit Gateway ID` dropdown and select `A4LTGW`
    ![alt text](image-5.png)
    * Select `VPN` for attachment type
    * Select `Existing` for Customer gateway
    * Click `Customer gateway ID` dropdown and select `ONPREM-ROUTER2`
    ![alt text](image-7.png)
    * Click `Dynamic (requires BGP)` for Routing options
    * Click `Enable Acceleration`
    * Click `Create transit gateway attachment`

* Move to `Site-to-Site VPN Connections` under `Virtual Private Network`

    For each of the connections, it will show you the `Customer Gateway Address` these match `ONPREM-ROUTER1` Public and `ONPREM-ROUTER2` Public

* Select the line which matches `Router1PubIP`
    ```
        Router1Public: 34.232.97.254
        Router2Public: 18.214.111.154
* Click `Download Configuration`
![alt text](image-8.png)
* Change vendor to `Generic`
![alt text](image-9.png)
* Click `Download`
* Rename this file to `CONNECTION1CONFIG.TXT`
* Repeat the process for connection 2. Select the line which matches **Router2PubIP**
* Click `Download Configuration`
Change vendor to `Generic`
Click `Download`
Rename this file to `CONNECTION2CONFIG.TXT`

### 2. POPULATE DEMO VALUE TEMPLATE WITH ALL CONFIG VALUES
    * There is a document in this folder called `DemoValueTemplate.md` - it contains instructions on how to extract all of the configuration variables you will need
    You will extract these from three locations

        * Outputs of the ONPREM CFN Stack
        * For Connection1, CONNECTION1CONFIG.TXT
        * For Connection2, CONNECTION2CONFIG.TXT

    Go ahead and populate that template using the instructions in the template

## Stage 3 -  IPSEC TUNNEL CONFIG
### 1. Move to EC2 console
* Click `Instances` on the hamburger menu
* Locate and select `ONPREM-ROUTER1`
* Right Click, select `Connect`
* Select `Session Manager`
* Click `Connect`
    * Type the following:
        * `sudo bash`
        * `cd /home/ubuntu/demo_assets/`
        * `nano ipsec.conf`

 
   *This is is the file which configures the IPSEC Tunnel interfaces over which our VPN traffic flows.*
   
   As we are connected to Router 1 - This configures the ones for ROUTER1 -> BOTH AWS Endpoints

* Replace the following placeholders with the real values in the `DemoValueTemplate.md` document


    1. `ROUTER1_PRIVATE_IP`
    2. `CONN1_TUNNEL1_ONPREM_OUTSIDE_IP`
    3. `CONN1_TUNNEL1_AWS_OUTSIDE_IP`
    4. `CONN1_TUNNEL1_AWS_OUTSIDE_IP`
    and
    5. `ROUTER1_PRIVATE_IP`
    6. `CONN1_TUNNEL2_ONPREM_OUTSIDE_IP`
    7. `CONN1_TUNNEL2_AWS_OUTSIDE_IP`
    8. `CONN1_TUNNEL2_AWS_OUTSIDE_IP`
    ![alt text](image-10.png)
    * `ctrl+o` to save, and `ctrl+x` to exit


* type `nano ipsec.secrets`
    
    *This file controls authentication for the tunnels*

    Replace the following placeholders with the real values in the `DemoValueTemplate.md` document

    1. `CONN1_TUNNEL1_ONPREM_OUTSIDE_IP`
    2. `CONN1_TUNNEL1_AWS_OUTSIDE_IP`
    3. `CONN1_TUNNEL1_PresharedKey`
    and
    4. `CONN1_TUNNEL2_ONPREM_OUTSIDE_IP`
    5. `CONN1_TUNNEL2_AWS_OUTSIDE_IP`
    6. `CONN1_TUNNEL2_PresharedKey`
    ![alt text](image-11.png)

    * `Ctrl+o` to save, and `Ctrl+x` to exit

* type `nano ipsec-vti.sh`
    *This script brings UP the IPSEC tunnel interfaces when needed*
    
    Replace the following placeholders with the real values in the `DemoValueTemplate.md` document
    
        `CONN1_TUNNEL1_ONPREM_INSIDE_IP` (ensuring the /30 is at the end)
        `CONN1_TUNNEL1_AWS_INSIDE_IP` (ensuring the /30 is at the end)
        `CONN1_TUNNEL2_ONPREM_INSIDE_IP` (ensuring the /30 is at the end)
        `CONN1_TUNNEL2_AWS_INSIDE_IP` (ensuring the /30 is at the end)

    * `Ctrl+o` to save, and `Ctrl+x` to exit

* `cp ipsec* /etc`
* `chmod +x /etc/ipsec-vti.sh`

Now all the configuration for Router1 IPSEC has been completed, lets restart the strongSwan service to bring them up.

* Type `systemctl restart strongswan` to restart strongSwan ... this should bring up the tunnels

* We can check these tunnels are up by running
`ifconfig`
* You should see `vti1` and `vti2` interfaces
![alt text](image-12.png)
* You can also check the connection in the AWS VPC Console ...the tunnels should be down, but IPSEC should be shown as UP after a few minutes.

### 2. CONFIGURE IPSEC TUNNELS FOR ONPREMISES-ROUTER2
Move to EC2 Console
https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:sort=instanceState
Click Instances on the left menu
Locate and select ONPREM-ROUTER2
Right Click => Connect
Select Session Manager
Click Connect

sudo bash
cd /home/ubuntu/demo_assets/
nano ipsec.conf

This is is the file which configures the IPSEC Tunnel interfaces over which our VPN traffic flows.
As we are connected to Router 2 - This configures the ones for ROUTER2 -> BOTH AWS Endpoints
Replace the following placeholders with the real values in the DemoValueTemplate.md document

    ROUTER2_PRIVATE_IP
    CONN2_TUNNEL1_ONPREM_OUTSIDE_IP
    CONN2_TUNNEL1_AWS_OUTSIDE_IP
    CONN2_TUNNEL1_AWS_OUTSIDE_IP
    and
    ROUTER2_PRIVATE_IP
    CONN2_TUNNEL2_ONPREM_OUTSIDE_IP
    CONN2_TUNNEL2_AWS_OUTSIDE_IP
    CONN2_TUNNEL2_AWS_OUTSIDE_IP

ctrl+o to save and ctrl+x to exit

nano ipsec.secrets

This file controls authentication for the tunnels
Replace the following placeholders with the real values in the DemoValueTemplate.md document

    CONN2_TUNNEL1_ONPREM_OUTSIDE_IP
    CONN2_TUNNEL1_AWS_OUTSIDE_IP
    CONN2_TUNNEL1_PresharedKey
    and
    CONN2_TUNNEL2_ONPREM_OUTSIDE_IP
    CONN2_TUNNEL2_AWS_OUTSIDE_IP
    CONN2_TUNNEL2_PresharedKey

Ctrl+o to save
Ctrl+x to exit

nano ipsec-vti.sh

This script brings UP the tunnel interfaces when needed
Replace the following placeholders with the real values in the DemoValueTemplate.md document

    CONN2_TUNNEL1_ONPREM_INSIDE_IP (ensuring the /30 is at the end)
    CONN2_TUNNEL1_AWS_INSIDE_IP (ensuring the /30 is at the end)
    CONN2_TUNNEL2_ONPREM_INSIDE_IP (ensuring the /30 is at the end)
    CONN2_TUNNEL2_AWS_INSIDE_IP (ensuring the /30 is at the end)

Ctrl+o to save
Ctrl+x to exit

cp ipsec* /etc
chmod +x /etc/ipsec-vti.sh

Now all the configuration for Router1 IPSEC has been completed, lets restart the strongSwan service to bring them up.

systemctl restart strongswan to restart strongSwan ... this should bring up the tunnels

We can check these tunnels are up by running
ifconfig
You should see vti1 and vti2 interfaces
You can also check the connection in the AWS VPC Console ...the tunnels should be down, but IPSEC should be shown as UP after a few minutes.
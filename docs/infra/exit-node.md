# Setup a VPN Exit Node

This guide will help you set up a CJDNS node with PKT wallet and the following services: You can follow the [steps](/infra/vpn-exit/) to set up the server or read more about the process and [services](/infra/vpn/) involved.

* [AnodeVPN server](https://github.com/anode-co/anodevpn-server){:target="_blank"}
* [IKEv2 Ipsec VPN server](https://github.com/hwdsl2/setup-ipsec-vpn){:target="_blank"}
* [OpenVPN server](https://ubuntu.com/server/docs/how-to-install-and-use-openvpn){:target="_blank"}
* [SNI proxy](https://github.com/dlundquist/sniproxy){:target="_blank"}


!!! danger "Requirements"

    * A server running debian based Linux (preferably Ubuntu 22.04)
    * Install docker


https://docs.docker.com/engine/install/ubuntu/ * Install jq

```console
sudo apt-get install jq
```

## Steps

* Create a data directory where the server configuration will be stored.

    ```console
    mkdir vpn_data
    ```

* Configure the server by running the following command:

    ```console
    docker run -it --rm -v $(pwd)/vpn_data:/data pkteer/pkt-server /configure.sh
    ```

* Configure various service by running the following command:

    ```console
    ./vpn_data/setup.sh
    ```

    The script will prompt you to set up various flags and values needed for setting up the services the first time.

* Run the server by running the following commands:

    ```console
    ./vpn_data/start-vpn.sh
    ```

    !!! info "NOTE"
        It can take a few minutes on the first run for the server to set up all the services.


## Monitoring the server

You can view the progress of the server by running:
```console
docker logs -f pkt-server
```

You can also check the status of all services by running:
```console
./vpn_data/status.sh
```

or using the AnodeVPN API:
```console
http://[server]:8099/api/0.4/server/status/
```

## Understanding the process
### Configuration

The `configure.sh` script is designed to set up and configure the server environment. Below is a detailed explanation of its functionality:

#### Initialization and Default Values:

The script initializes several flags and variables with default values: 

* __no_vpn_flag__: Indicates whether VPN should be disabled (default: false). 
* __cjdns_flag__: Indicates whether CJDNS should be enabled (default: true). 
* __with_pktd_flag__: Indicates whether PKTD should be enabled (default: false). 
* __pktd_passwd__: Stores the PKTD password (default: empty). 
* __pktd_user__: Stores the PKTD username (default: "x").

#### Configuration File Handling:

The script checks if the configuration file `/data/config.json` exists. If it does, it reads the existing configuration; otherwise, it copies a template configuration from `/server/config.json` to `/data/config`.json.

#### Configuration Synchronization:

If the configuration file exists, the script ensures that all fields from the template configuration are present in the existing configuration. Missing fields are added with their default values from the template.

#### Flag Parsing:
The script parses command-line arguments to set various flags: 

* __--no-vpn__: Disables VPN. 
* __--with-pktd__: Enables PKTD. 
* __--pktd-passwd=__: Sets the PKTD password.

#### Configuration Modification:
Based on the parsed flags, the script modifies the configuration: Sets the VPN exit status based on *no_vpn_flag*. Enables or disables PKTD based on *with_pktd_flag*. Generates a random PKTD password if none is provided. Updates the PKTD username and password in the configuration.

#### VPN Server Configuration:
Retrieves the PKT Wallet secret for the VPN server and ensures the `cjdroute.conf` configuration file is valid. If the file does not exist, it generates a new one and seeds it with the retrieved secret.

#### Security Configuration:
Modifies the cjdroute.conf file to set specific security parameters.

#### Script Deployment:
Finally it copies several utility scripts from the server directory to the data directory for further use.

### Initialization

The `init.sh` starts everytime the docker container is launched and is responsible for setting up and initializing various services and configurations on the server.

1. __Server Configuration Check__: It first checks if the server has already been configured by looking if cjdroute.conf exists. If the file is not found, the script exits.
2. __User Creation__: It creates two users, cjdns and speedtest. They are used for running the cjdns and speed-test (iperf3) services, respectively.
3. __Configuration Updates__: It reads the config.json configuration file and updates several configuration settings related to different services.
4. __PKT Wallet Initialization__: It starts the PKT Wallet service and checks if a wallet already exists. If the wallet exists, it attempts to unlock it. If the unlock request times out, it restarts the wallet service.
5. __Cjdns Service__: If cjdns is enabled, it sets up the necessary environment and starts the cjdns service. It also configures network settings and firewall rules to allow cjdns to function properly.
6. __PKTD Service__: If PKTD is enabled, it constructs and runs the command to start the PKTD service with the appropriate configuration.
7. __Network Interface Check__: It waits for tun0 network interface that is expected to be created by cjdroute to become available and then sets up firewall rules for network traffic.
8. __NFTables Initialization__: It initializes NFTables, which is a framework for packet filtering and network address translation.
9. __VPN Server__: If the VPN server is enabled, it starts the VPN server and sets up the pricing for the VPN service.
10. __Speed Test Service__: It sets up the environment for running speed tests and starts the necessary services.
11. __Cjdns Peers__: It adds peers for the cjdns network, the peers used are set in `/server/cjdnspeers.json` file.
12. __IKEv2 and OpenVPN__: If IKEv2 or OpenVPN are enabled, it runs the respective configuration scripts to set up these VPN services.
13. __Node Exporter__: It starts the Node Exporter service for Prometheus monitoring.
14. __SNI Proxy__: If SNI Proxy is enabled, it starts the SNI Proxy service.
15. __Cron Job for Payments__: It adds a cron job to handle payments on a weekly basis.
16. __Watchdog Service__: If cjdns is enabled, it starts a watchdog service to monitor and maintain the cjdns service the AnodeVPN server and other services depending on the configuration of the server.
17. __Keep-Alive__: Finally, it keeps the container running indefinitely by tailing `/dev/null`.

### Monitoring with watchdog

The `watchdog.sh` is monitoring the cjdroute service and if it stops it will restart it, when the cjdroute is restarted the AnodeVPN server is also restarted.

It will also check for the `pluto` service which is the IKEv2 Ipsec VPN server and if it stops it will restart it. Similarly for the `openvpn` service.

Finally the watchdog also checks the validity of vpnclients created by the AnodeVPN server and if their time has expired it will remove them.

### Checking the status

You can check the status of the services at any time either by running 

    ```console
    vpn_data/status.sh
    ```

or by using the AnodeVPN API:

    ```console
    http://[server]:8099/api/0.4/server/status/
    ```

Next to each service you will see the process id if that service is running, otherwise it will be `0`. e.g.

    {
        "hostname": "kraut2.pkteer.com",
        "pktwallet": 67,
        "cjdns": 82,
        "anodeserver": 114,
        "ikev2": 4012,
        "openvpn": 4135,
        "watchdog": 4116,
        "date_time": "2024-07-17 10:08:19"
    }


## Understanding the services and files

### Launching the server

The `vpn_data/start-vpn.sh` and `vpn_data/start.sh` scripts are designed to set up and run a VPN server using Docker. Here's a step-by-step explanation of what the script does:

* __Environment Setup__: It checks for the presence of necessary commands (jq, dirname, and docker). If any of these commands are missing, the script exits with an error message.
* __Directory Navigation__: It changes the working directory to the location of the script.
* __Cjdns Port Extraction__: It reads the cjdroute.conf file to extract the port number used by the cjdjns service. If the port number is not found in the expected format, it attempts to extract it using an alternative method.
* __Configuration Reading__: It reads the config.json file to get the region and city information. If either the region or city is not specified, the script exits with an error message.
* __Cjdns RPC Port Setup__: If the cjdns RPC (Remote Procedure Call) is enabled, it extracts the RPC port from the cjdroute.conf file and updates the configuration to expose the RPC port.
* __Docker Container Execution__: It runs a Docker container with various configurations:

Sets the timezone based on the region and city defined in `vpn_data/config.json`. Configures logging, network capabilities, and device access. Sets system control parameters for IPv6 and IPv4 forwarding. Maps several ports for different services: 

* reads the CJDNS port from the cjdroute.conf file and maps it to the host. 
* 5201 port for the speed-test (iperf3) service. 
* 64764 to the host for pktd service. 
* 443 for the SNI Proxy service. 
* 80 for the SNI Proxy service. 
* 500 for the IKEv2 Ipsec VPN server (pluto service). 
* 4500 for the IKEv2 Ipsec VPN server (pluto service). 
* 943 for the OpenVPN server. 
* 1194 for the OpenVPN server.

Mounts necessary directories for data persistency and configuration files. 

* /etc/openvpn to vpn_data/openvpn. 
* /server/vpnclients to vpn_data/vpnclients. 
* /data to vpn_data where the configuration files are stored, `cjdroute.conf` and `config.json` and others. 

Optionally maps the CJDNS RPC port if it is enabled. Runs the container in detached mode with elevated privileges. The script ensures that all necessary configurations are in place and starts the VPN server within a Docker container, making it ready for use.

### Cjdns

Cjdns is running on the server using a generated `cjdroute.conf` file. 

* The `cjdroute.conf` file is generated by the configure script 
* It is being launched by the init script which is run on the server start. 
* For persistency the file is stored in the `vpn_data` directory and used by the cjdns service. 

You can manually edit the file to add more cjdns peers.

!!! info "Note"
    Changing other parts of the configuration manually may end up breaking the service.

    * Wathdog is configured to keep cjdns running all the time. If the service stops, the watchdog will restart it.
    * If cjdroute is for some reason stuck or frozen you can kill it by running 
        
        `docker exec -it pkt-server killall cjdroute`
        
        and the watchdog will restart it.


### AnodeVPN Server

The server is running the AnodeVPN server to authorize clients and offer API access to the VPN services such as:

* Add domain to SNI proxy 

    `http://[server]:8099/api/0.4/server/domain/add/`

    ```console
    json { "domain": "example.com", "cjdnsIpv6": "fc00:0000:0000:0000:0000:0000:0000:0001" }
    ```

* Remove domain from SNI proxy 
    
    `http://[server]:8099/api/0.4/server/domain/remove/`

    ```console
    json { "domain": "example.com", "cjdnsIpv6": "fc00:0000:0000:0000:0000:0000:0000:0001" }
    ```

* Request new PKT address 

    `http://[server]:8099/api/0.4/server/premium/address/`

    ```console
    json {}
    ```

* Request new client VPN certificates

    `http://[server]:8099/api/0.4/server/vpnaccess/`

    ```console
    json { "address": "pkt1...." }
    ```


For more details see the [AnodeVPN](https://anode.co/#/){:target="_blank"}


### IKEv2 Ipsec VPN Server

The IKEv2 Ipsec VPN server is running on the server and is used to provide VPN services to clients with access to cjdns network.

For setting up the server the init script will launch the `vpn_configure.sh` if the `ikev2:enabled` flag is set to true, which will set up the server.

We also use the `ikev2.sh` script to add/remove clients through the AnodeVPN API. The files generated are copied in `server/vpnclients` directory which is mapped to `vpn_data/vpnclients/` on the docker host.

For more details look into the `setup-ipsec-vpn` documentation for configuring the server, managing clients and troubleshooting.

!!! info "NOTE"
    Unfortunately although IKEv2 clients can connect to the server from a Windows client and get VPN access, the clients are not able to access the CJDNS network. This is a known issue and we are working on a solution, for this reason we have added the OpenVPN server as an alternative for Windows users.


### OpenVPN Server

The openvpn is initialized by the init script if the `openvpn.enabled` flag is set to true in `config.json` and is used to provide VPN services to clients with access to cjdns network.

The server is configured using the `openvpn_configure.sh` which is used to generate the certificates needed.

Then for adding new clients the `createOpenVpnClient.sh` script is used by the AnodeVPN Server API to generate the client certificates and keys. The files are stored in the `vpn_data/vpnclients/` directory and can be used to connect to the server.

!!! info "NOTE"
    The OpenVPN server is running on the server and is used to provide VPN services to clients with access to cjdns network. This was added on top of the IKEv2 for Windows users to be able to access the cjdns network, but it can be used by any OpenVPN client on any platform.


### SNI Proxy

Proxies incoming HTTP and TLS connections based on the hostname contained in the initial request of the TCP session. This enables websites that are hosted on CJDNS network to become available via HTTPS name-based virtual hosting.

The SNI proxy will start if the `sniproxy.enabled` flag is set to true in the `config.json` file.

The sniproxy is using the `sniproxy.conf` file to route the requests to the correct server.

The server contains the default configuration for the sniproxy and is being edited by the `adddomain.sh` and `removedomain.sh` scripts which are used by the AnodeVPN API to add and remove domains respectively from the proxy.

For troubleshooting you can view the sniproxy logs. The access log is stored in `vpn_data/sniproxy-access.log` and the error log is stored in `vpn_data/sniproxy-error.log`.

### Setup a Domain Node
When you own a PKT vanity domain, it enables sovereignty for PKT websites. More information on how to set up a Domain Node will be available prior to Infrastructure Day on October 30, 2024.

### Setup a Nameserver Node
Packet Network has DNS (Domain Name System) that leverages the Packet Network to map domain names to cjdns IP addresses. More information on how to set up a Nameserver Node will be available in 2025.

### Setup a Route Server Node
The route server is used to route traffic on the Packet network. More information on how to set up a Route Server Node will be available in 2025.
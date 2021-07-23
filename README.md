# MQTT_VPN
VPN over MQTT

The idea of this project is to enable bidirectional IP connectivity, where it is not available otherwise, e.g. if one node is hidden behind (several layers of) NAT. This is the case in most private networks and also in mobile IP networks. Prerequisite is, that all connected nodes can reach a common MQTT broker. This allows you to "dial-in" into an IoT device sitting anywhere in the internet.

On all connected clients it sets up a simple (optionally encrypted) IP network where all clients connected to the same MQTT broker can communicate with each other via IP (IPv6 is not yet working). IP is tunneled via MQTT. On the nodes it creates a TUN interface, assignes an IP address, and publishes all packets to an MQTT topic mqttip/[dest IP address]. At the same time it subscribes to an MQTT topic mqttip/[own IP address]. This establishes a fully connected IP network. "Switching" is done via the pub/sub mechanism of the MQTT broker.

## Security
Encryption and authentication is implemented via a preshared password for all clients of this VPN, similar to a WiFi key. This password is a parameter of the interface initialization. It it hashed to a symmetic key used for encrypting and decrypting each packet. Using the "crypto_secretbox" of libnacl, a well-established crypto lib (https://nacl.cr.yp.to/ ), each message is now encrypted and authenticated as one coming from another client of the VPN network segment. Each encrypted packet includes a 192 bit random nonce, this guarantees that even identical packets will be encrypted differently.

With encryption in place, an adversary controlling the MQTT broker or the network can still observe the traffic pattern (who is when sending packets to whom) and packet lengths as well as replaying single packets, but he won't be able to decode any packet or send own packets.

## Linux

This Linux version can communicate with the Arduino and the ESP32 version above. Run this prog in background and use all standard network tools (including wireshark).

To build mqtt_vpn:
```
mkdir mqtt_vpn
cd mqtt_vpn
wget https://raw.githubusercontent.com/levkovigor/MQTT_VPN_ESP32/main/Makefile
wget https://github.com/levkovigor/MQTT_VPN_ESP32/blob/main/mqtt_vpn.c
make
```

If you start for example:
```
sudo ./mqtt_vpn -i mq0 -a 10.0.1.1 -b tcp://my_broker.org:1883 -k secret -d
```
You will see with "ifconfig" a new network interface "mq0" with address 10.0.1.1. It is connected via the broker to all other devices using the VPN over MQTT. Now you can reach an ESP8266 running the program above with
```
telnet 10.0.1.2 23
```

Usage:
```
mqtt_vpn -i <if_name> -a <ip> -b <broker> [-m <netmask>] [-n <clientid>] [-d]

-i <if_name>: Name of interface to use (mandatory)
-a <ip>: IP address of interface to use (mandatory)
-b <broker>: Address of MQTT broker (like: tcp://broker.io:1883) (mandatory)
-m <netmask>: Netmask of interface to use (default 255.255.255.0)
-u <username>: user of the MQTT broker
-p <password>: password of the MQTT broker user
-k <password>: preshared key for all clients of this VPN (no password = no encryption, default)
-6 <ip6>: IPv6 address of interface to use
-x <prefix>: prefix length of the IPv6 address (default 64)
-n <clientid>: ID of MQTT client (MQTT_VPN_<random>)
-t <ip>: IP address of a target to NAT
-d: outputs debug information while running
-h: prints this help text
```

Requires the Paho MQTT C Client Library (https://www.eclipse.org/paho/files/mqttdoc/MQTTClient/html/index.html ) and the libnacl for the crypto operations.

## Paho MQTT C Client Library Installation
```
git clone https://github.com/eclipse/paho.mqtt.c
make
sudo make install
```

## libnacl Installation
```
sudo apt-get install -y libnacl-dev
```

## Thanks
- martin-ger for MQTT_VPN (https://github.com/martin-ger/MQTT_VPN )

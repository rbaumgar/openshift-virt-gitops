## Delete device

nmcli dev delete br1

## Show devices

nmcli dev show [device]

## Show Connections

nmcli connection show

# Create bridge and add eth0 to it
nmcli con add type bridge ifname br1 stp no
nmcli con add type bridge-slave ifname eth1 master br1

## Enable the bridge
nmcli con up bridge-br1
nmcli con up bridge-slave-eth1

## Disable the bridge
nmcli con down bridge-br1
nmcli con down bridge-slave-eth1
# multipass
multipass launch -n vm00 -m 2Gb -d 50Gb -c 2
multipass launch -n vm01 -m 2Gb -d 50Gb -c 2
multipass launch -n vm02 -m 2Gb -d 50Gb -c 2

multipass launch -n vm03 -m 2Gb -d 50Gb -c 2
multipass launch -n vm04 -m 2Gb -d 50Gb -c 2
multipass launch -n vm05 -m 2Gb -d 50Gb -c 2

# overlay vxlan
multipass list --format=json | jq '.list[] | {"ip": ( .ipv4 | select( .[] | test("^10\\.0") | not) | .[0] ), "name": .name} | select(.name | test("^vm"))' > ips.json

VM00IP=$(cat ips.json | jq -r 'select(.name == "vm00").ip')
VM01IP=$(cat ips.json | jq -r 'select(.name == "vm01").ip')
VM02IP=$(cat ips.json | jq -r 'select(.name == "vm02").ip')
GUESTDEVNAME=enp0s1

multipass exec vm00 -- sudo ip link add vxlan100 type vxlan id 100 dev $GUESTDEVNAME dstport 4789 local $VM00IP remote $VM01IP 
multipass exec vm00 -- sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst $VM02IP

multipass exec vm01 -- sudo ip link add vxlan100 type vxlan id 100 dev $GUESTDEVNAME dstport 4789 local $VM01IP remote $VM00IP 
multipass exec vm01 -- sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst $VM02IP

multipass exec vm02 -- sudo ip link add vxlan100 type vxlan id 100 dev $GUESTDEVNAME dstport 4789 local $VM02IP remote $VM01IP 
multipass exec vm02 -- sudo bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst $VM00IP

multipass exec vm00 -- sudo ip link set vxlan100 up
multipass exec vm01 -- sudo ip link set vxlan100 up
multipass exec vm02 -- sudo ip link set vxlan100 up

multipass exec vm00 -- sudo ip addr add 10.0.0.10/24 dev vxlan100
multipass exec vm01 -- sudo ip addr add 10.0.0.11/24 dev vxlan100
multipass exec vm02 -- sudo ip addr add 10.0.0.12/24 dev vxlan100

# tc

multipass exec vm00 -- sudo tc qdisc del dev vxlan100 root netem delay 5ms 5ms loss 1% 25%

multipass exec vm00 -- sudo tc qdisc add dev vxlan100 root netem delay 5ms 5ms loss 0% 0%
multipass exec vm01 -- sudo tc qdisc add dev vxlan100 root netem delay 5ms 5ms loss 0% 0%
multipass exec vm02 -- sudo tc qdisc add dev vxlan100 root netem delay 5ms 5ms loss 0% 0%



multipass exec vm00 -- sudo tc qdisc show dev vxlan100
multipass exec vm02 -- sudo tc qdisc show dev vxlan100

sudo tcpdump -i vxlan100 icmp -tttt

# second overlay

multipass list --format=json | jq '.list[] | {"ip": ( .ipv4 | select( .[] | test("^10\\.0") | not) | .[0] ), "name": .name} | select(.name | test("^vm"))' > ips.json

VM03IP=$(cat ips.json | jq -r 'select(.name == "vm03").ip')
VM04IP=$(cat ips.json | jq -r 'select(.name == "vm04").ip')
VM05IP=$(cat ips.json | jq -r 'select(.name == "vm05").ip')
GUESTDEVNAME=enp0s1

multipass exec vm03 -- sudo ip link add vxlan200 type vxlan id 200 dev $GUESTDEVNAME dstport 4790 local $VM03IP remote $VM04IP 
multipass exec vm03 -- sudo bridge fdb append 00:00:00:00:00:00 dev vxlan200 dst $VM05IP

multipass exec vm04 -- sudo ip link add vxlan200 type vxlan id 200 dev $GUESTDEVNAME dstport 4790 local $VM04IP remote $VM03IP 
multipass exec vm04 -- sudo bridge fdb append 00:00:00:00:00:00 dev vxlan200 dst $VM05IP

multipass exec vm05 -- sudo ip link add vxlan200 type vxlan id 200 dev $GUESTDEVNAME dstport 4790 local $VM05IP remote $VM04IP 
multipass exec vm05 -- sudo bridge fdb append 00:00:00:00:00:00 dev vxlan200 dst $VM03IP

multipass exec vm03 -- sudo ip link set vxlan200 up
multipass exec vm04 -- sudo ip link set vxlan200 up
multipass exec vm05 -- sudo ip link set vxlan200 up

multipass exec vm03 -- sudo ip addr add 10.0.1.13/24 dev vxlan200
multipass exec vm04 -- sudo ip addr add 10.0.1.14/24 dev vxlan200
multipass exec vm05 -- sudo ip addr add 10.0.1.15/24 dev vxlan200

# connection

multipass list --format=json | jq '.list[] | {"ip": ( .ipv4 | select( .[] | test("^10\\.0") | not) | .[0] ), "name": .name} | select(.name | test("^vm"))' > ips.json

VM02IP=$(cat ips.json | jq -r 'select(.name == "vm02").ip')
VM03IP=$(cat ips.json | jq -r 'select(.name == "vm03").ip')
GUESTDEVNAME=enp0s1

multipass exec vm02 -- sudo ip link add vxlan300 type vxlan id 300 dev $GUESTDEVNAME dstport 4791 local $VM02IP remote $VM03IP 
multipass exec vm03 -- sudo ip link add vxlan300 type vxlan id 300 dev $GUESTDEVNAME dstport 4791 local $VM03IP remote $VM02IP 

multipass exec vm02 -- sudo ip link set vxlan300 up
multipass exec vm03 -- sudo ip link set vxlan300 up

multipass exec vm02 -- sudo ip addr add 10.0.2.12/24 dev vxlan300
multipass exec vm03 -- sudo ip addr add 10.0.2.13/24 dev vxlan300

multipass exec vm00 -- sudo ip route add 10.0.1.0/24 via 10.0.0.12
multipass exec vm01 -- sudo ip route add 10.0.1.0/24 via 10.0.0.12
multipass exec vm02 -- sudo ip route add 10.0.1.0/24 via 10.0.2.13
multipass exec vm03 -- sudo ip route add 10.0.0.0/24 via 10.0.2.12
multipass exec vm04 -- sudo ip route add 10.0.0.0/24 via 10.0.1.13
multipass exec vm05 -- sudo ip route add 10.0.0.0/24 via 10.0.1.13

multipass exec vm02 -- sudo sysctl -w net.ipv4.ip_forward=1
multipass exec vm03 -- sudo sysctl -w net.ipv4.ip_forward=1

tcpdump -i vxlan100
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

sudo ip link add vxlan100 type vxlan id 100 dev eth0 dstport 4789 local 10.0.0.1 \
  remote 10.0.0.2 remote 10.0.0.3

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

multipass exec vm00 -- sudo tc qdisc add dev vxlan100 root netem delay 40ms 10ms loss 1% 25%
multipass exec vm00 -- sudo tc qdisc del dev vxlan100 root netem delay 40ms 10ms loss 1% 25%
multipass exec vm00 -- sudo tc qdisc add dev vxlan100 root netem delay 5ms 5ms loss 1% 25%


multipass exec vm00 -- sudo tc qdisc show dev vxlan100
multipass exec vm02 -- sudo tc qdisc show dev vxlan100

sudo tcpdump -i vxlan100 icmp -tttt

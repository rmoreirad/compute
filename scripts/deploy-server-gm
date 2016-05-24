#!/bin/bash -x
function destroy_vms {
	for i in `cloudmonkey -d table list virtualmachines filter=id | awk 'NR>4 { print $2}' | xargs `
	do
		cloudmonkey destroy virtualmachine id=$i
	done
}

function destroy_ips {
	for i in `cloudmonkey -d table list publicipaddresses filter=id  | awk 'NR>4 { print $2}' | xargs`
	do
		cloudmonkey disassociate ipaddress id=$i
	done
}

CLI="cloudmonkey -d table"

#Service Offerings
SO_EXTRA_PEQUENA=e1edb14f-2a09-48cb-a67f-7009566e1ab8
SO_PEQUENA=fa09b37a-e27a-4d7b-8ca4-98d5143bc943
SO_MEDIA=a021fd71-0998-4dae-b291-46e178cbbbf3
SO_GRANDE=5a21d3b3-9496-4bc1-a84b-d5247783b48c
SO_EXTRA_GRANDE=72df17a3-10eb-45e0-8184-51ac09701219

#Zones
Z_MANAUS=df33046e-ab2a-429c-8653-09d869a9d7aa
Z_RECIFE=0d89768b-bdf5-455e-b4fd-91881fa07375

#Networks
N_MANAUS=b6c1a248-e089-4533-b2af-8ea8a28eddc7
N_RECIFE=e413a713-4f82-499f-975d-584be91bd2fb

#ssh keypair
SSH_KEYPAIR=guimaluf


#MY CIDR
CIDR=150.164.3.220/32


ZONE=$Z_MANAUS
NETWORK=$N_MANAUS
TEMPLATE_ID=c686e11d-b2a8-4a22-81e4-3cd02ea1c557 #Ubuntu 14.04 64bit
SERVICE_OFFERING=$SO_GRANDE

if [ "$1" = 'clean' ]; then
  echo 'Destroying vms...'
  destroy_vms

  echo 'Destroying ips...'
  destroy_ips
  echo '[jmeter]' > hosts
  exit 0
fi

for i in `seq $1`
do
  if [ $(( $i % 1 )) -eq 0 ] ; then
    ZONE=$Z_MANAUS
    NETWORK=$N_MANAUS
  else
    ZONE=$Z_RECIFE
    NETWORK=$N_RECIFE
  fi

  echo Deploying vm...
  DEPLOY=`$CLI deploy virtualmachine templateid=${TEMPLATE_ID} serviceofferingid=${SERVICE_OFFERING} zoneid=${ZONE} keypair=$SSH_KEYPAIR | grep -e'^id\ =' -e '^password\ =' | xargs`
  VM_ID=`echo $DEPLOY | awk '{print $3}'`
  VM_PASSWD=`echo $DEPLOY | awk '{print $6}'`

  echo Associating IP address...
  ASSOCIATE=`$CLI associate ipaddress networkid=${NETWORK} | grep -e '^id\ =' -e '^ipaddress\ =' | xargs`
  PUB_IP_ID=`echo $ASSOCIATE | awk '{print $3}'`
  PUB_IP=`echo $ASSOCIATE | awk '{print $6}'`

  echo Enable static NAT...
  $CLI enable staticnat ipaddressid=${PUB_IP_ID} virtualmachineid=${VM_ID}

  echo Firewall ssh rule...
  $CLI create firewallrule cidrlist=${CIDR} startport=22 endport=22 protocol=tcp ipaddressid=${PUB_IP_ID}
  #$CLI create firewallrule cidrlist=0.0.0.0/0 startport=1024 endport=65535 protocol=tcp ipaddressid=${PUB_IP_ID}
  sleep 30

  sshpass -p$VM_PASSWD ssh-copy-id root@$PUB_IP
  echo "sshpass -p$VM_PASSWD ssh-copy-id root@$PUB_IP " >> sshpass

  echo "cloustack${RANDOM} ansible_host=${PUB_IP}" >> hosts
  REMOTE_HOSTS="${PUB_IP},${REMOTE_HOSTS}"
done

echo REMOTE_HOSTS $REMOTE_HOSTS
ansible-playbook jmeter.yml -i hosts

exit 0


获取SR-IOV PF与VF关系

```bash
#!/bin/bash
function pf_vf(){
  echo "<=============>PF:$1<=============="
  echo "`lspci|grep $(ls -l /sys/class/net/$1/ |grep device |cut -d '/' -f 4|sed 's/0000://g')` --> $1"
  echo " VF:"
  eth_dev=`ls /sys/class/net/$1/device/virtfn* -l | cut -d ">" -f 2 |cut -d "/" -f 2`
  for i in $eth_dev; 
  do 
    vf_name=`ls /sys/bus/pci/devices/$i/net 2>&1`;
    if  [ $? -eq 0 ]; then
      echo " |_ `lspci|grep $(echo $i|sed 's/0000://g')` --> $vf_name"; 
      echo "   |_ _ $(ip link show |grep -w $vf_name -A1 |grep -v $vf_name|awk '{print $2}')";
    else
      echo "$i has bound by vm !"
    fi
  done
}
for i in $(ip link show |grep mq|cut -d ':' -f 2);do pf_vf $i;done
```




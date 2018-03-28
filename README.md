# GW_Check
# Bash-script for checking IP-addresses of SIP-equipment and change it in Asterisk sip.conf automatically
#!/bin/bash
##### VARIABLES
GW_SIP_INFO=/root/GW_SIP_INFO #Temporary file
FILE=/root/GW_INFO #Destination of the file with GW's MAC-adresses
SIP_CONF=/etc/asterisk/sip.conf
SIP_BACKUP=/etc/asterisk/sip_backup.conf
COUNT=0
##### END OF VARIABLES
##### FUNCTIONS
# create_GW_INFO    
# Create GW_INFO file and start fill_GW_INFO function
# fill__GW_INFO    # Parse sip.conf and write information in GW_INFO file once. Use with correct sip.conf !!!
# create_fill_GW_INFO   # Parse sip.conf and refresh information in GW_SIP_INFO file (tmp file with info from sip.conf)
##### END OF FUNCTIONS
         GW_quantity=$(grep -c "\[G....." $SIP_CONF )
         echo "GW counted, quantity = $GW_quantity"
##### Parse sip.conf function
function create_GW_INFO {
                rm $FILE
                touch $FILE
                echo "File GW_INFO was removed and created again. Filling GW_INFO in progress"
                echo -e "\n" >> $FILE
                fill_GW_INFO
                echo "File GW filled"
                cat $FILE
}
function fill_GW_INFO {
                while read LINE
                        do
                            if [[ $LINE == "[G"* ]];
                             then
                              Peer=0
                              Peer=$LINE
                              let "COUNT = COUNT + 1"
                                 sed -i "${COUNT}a\ $Peer-" $FILE
                            fi
                            if [[ $LINE == "host"* ]];
                                 then
                                 GW_IP=0
                                 GW_IP=`echo "$LINE" | cut -d= -f2`
                GW_MAC=`echo $(nmap $GW_IP) | awk -F"MAC Address: " '{print $2}' | grep -v :38 | awk '{print $1}'`
                                Peer=$(echo $Peer  | sed -r 's/(\[|\])/\\\1/g');
sed -i "s/${Peer}-/${Peer}-${GW_IP}_${GW_MAC}/" $FILE
                            fi
 done < $SIP_CONF
}
function fill_GW_SIP {
 rm $GW_SIP_INFO
 touch $GW_SIP_INFO
                echo "File GW_SIP_INFO was removed and created again. Filling GW_SIP_INFO in progress"
                echo -e "\n" >> $GW_SIP_INFO
                GCOUNT=0
               while read LINE
                        do
                            if [[ $LINE == "[G"* ]];
                             then
                             GPeer=0
                             GPeer=$LINE
                              let "GCOUNT = GCOUNT + 1"
                                 sed -i "${GCOUNT}a\ $GPeer-" $GW_SIP_INFO
                            fi
                            if [[ $LINE == "host"* ]];
                                 then
                                 GGW_IP=0
                                 GGW_IP=`echo "$LINE" | cut -d= -f2`
                               GPeer=$(echo $GPeer  | sed -r 's/(\[|\])/\\\1/g');
                                sed -i "s/${GPeer}-/${GPeer}-${GGW_IP}/" $GW_SIP_INFO
                            fi
 done < $SIP_CONF
}

if test -f $FILE # Use "!" after "test" for test mode
        then
         echo "File GW_INFO exists"
         fill_GW_SIP
        else
        fill_GW_SIP
        create_GW_INFO
fi
##### RASPBERRY IP-ADRESS $SBO_IP
SBO_IP=`ip -4 -o address show eth0 | awk '{print $4}' | cut -d '/' -f 1`
echo "SBO IP - $SBO_IP"
##### SCAN NMAP AND COMPARE MAC AND IP WITH GW_INFO
NMAC=`nmap -sP $SBO_IP/24`
while read LINE
 do
  GW_Peer=`echo $LINE | cut -d '-' -f 1`
  GW_MAC=`echo $LINE | cut -d '_' -f 2`
  GW_IP_REAL=`echo $NMAC |awk -F $GW_MAC '{print $1}' |awk -F "for" '{print $NF}' |awk -F " Host" '{print $1}' |cut -d ' ' -f 2 `
 GWPeer=$(echo $GW_Peer  | sed -r 's/(\[|\])/\\\1/g');
 GW_IP_FILE=`cat $GW_SIP_INFO | grep $GWPeer | cut -d '-' -f 2 | cut -d '_' -f 1`
 echo $GW_IP_FILE
  if [ "$GW_IP_REAL" = "$GW_IP_FILE" ]
   then
    echo "GW" $GW_Peer "IP matched, nothing to do. SBO IP =" $SBO_IP
   else
if test -f $SIP_BACKUP
        then
         rm /etc/asterisk/sip_backup.conf
cp /etc/asterisk/sip.conf /etc/asterisk/sip_backup.conf
echo "Backup file: /etc/asterisk/sip_backup.conf"
        else
cp /etc/asterisk/sip.conf /etc/asterisk/sip_backup.conf
echo "Backup file: /etc/asterisk/sip_backup.conf"
fi
  echo "GW" $GW_Peer "IP mismatch. Current GW IP =" $GW_IP_REAL ".IP was" $GW_IP_FILE ".Changing in sip.conf"
   sed -i "s/host=$GW_IP_FILE/host=$GW_IP_REAL/" $SIP_CONF
   asterisk -x 'sip reload'
  fi
done < $FILE


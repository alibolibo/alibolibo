# SNMP Traps

The SNMP Traps program consists on parsing the trap messages sent by agents to translate them to useful information. 

## Getting Started

These instructions will get you the right setting for development and testing purposes

### Prerequisites

First, net-snmp needs to be [installed](http://net-snmp.sourceforge.net/docs/INSTALL.html). Specially the snmptrapd program. 

Then the snmptrapd program has to be configured. For that, the file /etc/default/snmptrapd needs to have the next lines so that the traps functionality is enabled and also to store all traps in /var/log/traps.log

```
TRAPDRUN=yes
TRAPDOPTS='-Lf /var/log/traps.log -Lsd -p /run/snmptrapd.pid'
```

Then to configure the trap receiver, the file /etc/snmp/snmptrapd.conf needs to have the following line so that it receives traps with the community "public".

```
authCommunity log,execute,net public
```

Last, you need the traps to be sent by agents. To make it easier to do a test, the script script_trap.sh sends differents traps at different rates. Changing the snmptrap commands in the scripts, the traps can be also send from another computer and not only localhost.

## Running the tests

First, the snmptrapd has to be initiated:

```
service snmptrapd start
```

Now that it is ready to receive traps, the main program has to be started:

```
./test_snmp  -i /var/log/traps.log -o output -c http://admin:admin@localhost:5984/ -tw 10 -tr 1
```

* -i to select the input file where the traps are stored.
* -o to provide the name of the output file. That name will be followed by "_timestamp.csv".
* -c to provide the couchdb location. If not provided, it will be "http://admin:admin@localhost:5984/".
* -tw to provide the time interval in seconds in which a new output file with a new timestamp is created and the previous file is rename to "output_timestamp.finished". If not provided, it will be 60.
* -tr to provide the time interval in seconds in which the input file is checked to see if it has new traps. If not provided, it will be 10. 

This test is useful not only to see if everything works but also to check that it parses the traps fast enough by changing the script that sends the traps to increase the rate and checking the output files.

The couchDB is used to access the mib.json file that contains all OIDs and its translations. So in case the snmptrapd couldn't translate one of the OIDs then the program would access to couchDB to translate it. It is important that currently, the program asks for the OID starting by a digit (1.3.6), and not a dot (.1.3.6). 


## Documentation

### Traps content

All traps, as shown on /var/log/traps.log have the same format. They have two lines separated by \n. The first line provides the date and time when the trap was received and also de Agent IP that sent the trap.

```
2018-06-08 21:07:46 localhost [UDP: [127.0.0.1]:35831->[127.0.0.1]:162]:
```

The second line contains a sequence of OIDs variables, its type and its value. All OIDs are separated by tabs. 

```
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (24004) 0:04:00.04    SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkUp    IF-MIB::ifIndex.3 = INTEGER: 3    IF-MIB::ifAdminStatus.3 = INTEGER: up(1)    IF-MIB::ifOperStatus.3 = INTEGER: up(1)    SNMPv2-MIB::snmpTrapEnterprise.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
```

The following is an analysis of the traps content that are sent by the agents to the manager. It is an analysis of the traps as they are stored on /var/log/traps.log by snmptrapd.

First, the traps have been divided in this project based on how they are configured on the agent that sends them. The traps are configured on /etc/snmp/snmpd.conf and they can be generic traps and not generic traps.

#### Generic traps

Standard generic traps are: coldStart, warmStart, linkDown, linkUp, authenticationFailure, egpNeighborLoss. To analyse them, first there is the line at the snmpd.conf file which monitorizes to throw the trap and then the trap itself as it's stored in /var/log/traps.log.

Line on snmpd.conf: *linkUpDownNotifications  yes*

```
2018-06-08 21:07:46 localhost [UDP: [127.0.0.1]:35831->[127.0.0.1]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (24004) 0:04:00.04    SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkUp    IF-MIB::ifIndex.3 = INTEGER: 3    IF-MIB::ifAdminStatus.3 = INTEGER: up(1)    IF-MIB::ifOperStatus.3 = INTEGER: up(1)    SNMPv2-MIB::snmpTrapEnterprise.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
```
Content:

* Date and time
* Agent IP address
* DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (24004) (sysTimeStamp in centiseconds)
* SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkUp (identification of the notification currently being sent). Always has the format: SNMPv2-MIB::snmpTrapOID.0 = OID: XXXX::xxx. It may not be sent if snmp version 1.
* OIDs extra: IF-MIB::ifIndex.3, IF-MIB::ifAdminStatus, IF-MIB::ifOperStatus.
* SNMPv2-MIB::snmpTrapEnterprise (identification of the enterprise associated with the trap currently being sent)
* It doesn't contain the name of the interface. A snmpwalk could be made to get the name, as the IF-MIB::ifIndex is always sent.

The rest of the generic traps have more or less the same format and the same fields.

#### Not generic traps

These traps always are monitored on snmpd.conf starting with "monitor".

Line on snmpd.conf: *monitor -r 10 existencia 1.3.6.1.2.1.2.2.1.8*

```
2018-06-09 04:01:37 localhost [UDP: [127.0.0.1]:39921->[127.0.0.1]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (6) 0:00:00.06    SNMPv2-MIB::snmpTrapOID.0 = OID: DISMAN-EVENT-MIB::mteTriggerFired    DISMAN-EVENT-MIB::mteHotTrigger.0 = STRING: existencia    DISMAN-EVENT-MIB::mteHotTargetName.0 = STRING:     DISMAN-EVENT-MIB::mteHotContextName.0 = STRING:     DISMAN-EVENT-MIB::mteHotOID.0 = OID: IF-MIB::ifOperStatus.3    DISMAN-EVENT-MIB::mteHotValue.0 = INTEGER: 1
```

Content:
* Date and time
* Agent IP address
* DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (24004) (sysTimeStamp in centiseconds)
* SNMPv2-MIB::snmpTrapOID.0 = OID: (it's always DISMAN-EVENT-MIB::mteTriggerFired when the trap is a not generic trap) 
* DISMAN-EVENT-MIB::mteHotTrigger (name as shown on snmpd.conf)
* DISMAN-EVENT-MIB::mteHotTargetName (The SNMP Target MIB's snmpTargetAddrName related to the notification.)
* DISMAN-EVENT-MIB::mteHotContextName (context name related to the notification)
* DISMAN-EVENT-MIB::mteHotOID (OID being monitored)
* DISMAN-EVENT-MIB::mteHotValue (OID's value)

 All not generic traps are very similar. There could be extra OIDs at the end of the trap which coudl have extra information for the manager related to the origin of the trap. Those OIDs are also parsed by the program


## Future Work

All these traps are configured on snmpd.conf on the Agent. The future work would be to enable remote trap configuration, which means to enable the configuration from the manager by snmpsets so that the snmpd.conf is not changed and the traps can be created and deleted from the manager. This comes with some issues like that enabling snmpset on the agents with snmp version 2 could result on security problems because the community is not enough. One of the solutions is to use snmp version 3.

So, the way it works is basically sending snmpset messages from the manager to the agent to set the exact same OIDs that are setted when a new trap configuration is made on snmpd.conf. Those OIDs are the following:

* snmpTargetAddrTable
* mteEventNotificationTable
* dismanEventMIB

To learn how to configure the traps through snmpsets, first configure the traps through snmpd.conf and then check what changes have been made on these OIDs and try to mimic those configurations.



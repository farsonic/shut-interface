# shut-interface
1) Define a firewall filter that specifies the permitted source IP address ranges and also has a discard policy that will create a SYSLOG message for any traffic that doesn’t match these permitted source IP Addresses.  

set firewall family ethernet-switching filter allow-specific-subnets-only term allowed-ip-ranges from source-address 192.168.0.0/24
set firewall family ethernet-switching filter allow-specific-subnets-only term allowed-ip-ranges from source-address 192.168.1.0/24
set firewall family ethernet-switching filter allow-specific-subnets-only term allowed-ip-ranges from source-address 192.168.2.0/24
set firewall family ethernet-switching filter allow-specific-subnets-only term allowed-ip-ranges from source-address 192.168.3.0/24
set firewall family ethernet-switching filter allow-specific-subnets-only term allowed-ip-ranges then accept
set firewall family ethernet-switching filter allow-specific-subnets-only term alert then discard
set firewall family ethernet-switching filter allow-specific-subnets-only term alert then syslog

Any traffic that matches the denied entry of the firewall filter will generate a SYSLOG message that looks like this, notice that there is “PFE-FW-SYSLOG-ETH:” at the start of this log line, this will be there for any event that matches this. It would also be different if it was attached to an IP interface, but for my testing it is associated with an Ethernet interface. 

Sep 28 06:29:21  EX2200-12C fpc0 PFE_FW_SYSLOG_ETH: FW: ge-0/0/5.0   D ff:ff:ff:ff:ff:ff 08:81:f4:a8:da:09 0806 (1 packets) 

2) Once the firewall filter is created it needs to be assigned to one or all of the access interfaces facing the clients on the network. This is configured as follows; 
    With this applied your ethernet interface is going to monitor every source IP address and permit it if it matches the source-address entries. If it does not match then it will generate a SYSLOG message (as above) and discard the packet. 

admin@EX2200-12C> show configuration interfaces ge-0/0/5  
unit 0 {
    family ethernet-switching {
        filter {
            input allow-specific-subnets-only;
        }
    }
}

3) Now we need to configure an event entry that will react anytime it sees a PFE-FW-SYSLOG-ETH appear in the local syslog file. The following entry specifies exactly what to do if we see such a log entry appear. 

admin@EX2200-12C> show configuration event-options  
policy shut-interface {
    events pfe_fw_syslog_eth;
    attributes-match {
        pfe_fw_syslog_eth.count matches 1;
    }
    then {
        event-script shut-interface.slax {
            arguments {
                interface "{$$.interface-name}";
            }
        }
        raise-trap;
    }
}
event-script {
    file shut-interface.slax;
}

The above configuration triggers the “shut-interface.slax” script which resides in /var/db/scripts/event on the EX-Series switch. It extracts the interface-name from the SYSLOG entry and passes this as an argument called $interface to the script. It also generates a SNMP Trap which contains all the fields of the SYSLOG entry. 

4) We place the following script into /var/db/scripts/event. Notice how this is also specified/configured in the event-options section of the configuration above. If this wasn’t there then it wouldn’t be executed.

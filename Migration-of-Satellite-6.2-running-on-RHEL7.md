# Migration of Satellite 6.2 running on RHEL 7 to macro container

### Goal
To migrate an existing machine (virtual or physical) running RHEL 7 with installed Satellite 6 into a container which will be deployed and run on a target machine running Docker. The source machine must work without any issues as well as the target container when the migration process is done. 

**Note:** _No ports used by Satellite can be moved to different port numbers since all hosts registered with satellite would have issues._

### Source application configuration specifics
External DNS was used for translating satellite hostname to IP.

### Target application configuration specifics
/etc/hosts has to have a record translating old hostname to localhost ip in order to make the satellite internal services work properly.

`echo 127.0.0.1 satellite.example.com >> /etc/hosts`

After the migration is done, the old DNS must be updated to point to a new IP, as well as the /etc/hosts if required.

### Result
Test **met** defined expectations:
* Container is running after the migration
* Satellite is accessible using GUI and through yum as client
* IP address is the one of the host
* The data is stored in a directory on the host, container was started off that directory instead of an images

### Issues
There were observed stall issues during the migration process caused by XFS filesystem CLOSE operation implementation when filesystem was frozen. XFS driver can still write updates while the fs is being frozen, therefore not even read only operation could be closed. This could be solved by setting `--freeze-fs f`. 

/etc/hosts in the container must be updated after each start of the container

### Known limitations
Built-in DNS and DHCP will not work because LeApp tool v0.1 doesn’t support UDP protocol. As a result of this the machine provisioning will not work.
If target is running Cockpit, the port must be reallocated from 9090, because this port is also used by satellite.


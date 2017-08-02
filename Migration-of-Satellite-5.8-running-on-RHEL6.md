# Migration of Satellite 5.8 running on RHEL 6 to macro container

### Goal
To migrate an existing machine (virtual or physical) running RHEL 6 with installed Satellite 5 into a container which will be deployed and run on a target machine running Docker. The source machine must work without any issues as well as the target container when the migration process is done.

**Note:** _No ports used by Satellite can be moved to different port numbers since all hosts registered with satellite would have issues._


### Source application configuration specifics
Embedded PostgreSQL database

External database can be used with some limitations, see Known limitations for details

External DNS was used for translating satellite hostname to IP

### Target application configuration specifics
/etc/hosts has to have a record translating old hostname to localhost ip in order to make the satellite internal services work properly.

`echo 127.0.0.1 satellite.example.com >> /etc/hosts`

After the migration is done, the old DNS records must be updated to point to a new IP.

### Result
Test **met** defined expectations:
* Container is running after the migration
* Satellite is accessible using GUI and through yum as client
* IP address is the one of the host
* The data is stored in a directory on the host, container was started off that directory instead of an images

### Issues
/etc/hosts in the container must be updated after each start of the container.

### Known limitations
Built-in DNS and DHCP will not work because LeApp tool v0.1 doesn’t support UDP protocol. As a result of this is that the provisioning of machines will not work.

Only one instance must be running in a time, if there is external database used. This means, either source machine, or container can run, not both at the same time. It is strongly recommended to run rhn-satellite stop before the migration procedure.

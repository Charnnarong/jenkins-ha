# High availability Jenkins Master

This project was created for fun in order to learn how we can apply HA for Jenkins Master.

**Remark!** 

Open issue: 
	
	crmd[13763]:    error: Failed to retrieve meta-data for ocf:linbit:drbd
	crmd[13763]:  warning: No metadata found for drbd::ocf:linbit: Input/output error (-5)
	crmd[13763]:    error: No metadata for linbit::ocf:drbd
	crmd[13763]:    error: Failed to retrieve meta-data for ocf:linbit:drbd
	crmd[13763]:  warning: No metadata found for drbd::ocf:linbit: Input/output error (-5)
	crmd[13763]:    error: No metadata for linbit::ocf:drbd

Though drbd doesn't work properly for meta-data issue i.e when it's managed by pcs, but we might have to substitute the RAID 1 approach
with network file sharing and offload data replication to NFS itself. 


In the mean time , you can comment out `drdb` role in provisionHaSystem.yml and virtual IP and HTTP server will works as usual. 


**TODO**: 
	- Change HTTPD to Tomcat and deploy sample project.
	- Simulate NFS 
	
## Common tasks

Check Peacemaker version

	rpm -qa|grep -i pacemaker 
	
Check pcs status

	pcs status
	
Check startup error 

	journalctl | grep -i error
	
And it may logs into 
	
	/var/log/messages

Show Cluster configuration in xml

	pcs cluster cib
	
Check the validity of the configuration

	crm_verify -L -V
	
Available resource standards

	pcs resource standards
	


👉️NOTE: THERE IS A DESCRIPTION BELOW FOR EACH COMMAND

steps taken to a condition when slot change happens in a server for ceph disk

	1. systemctl stop ceph-osd@4.service
	
	2. sudo lvs -o lv_name,vg_name,lv_uuid

	slot changed here
	
	3. sudo vgchange -an ceph-4152b049-a23c-42ae-9e75-1a50ae887905
	
	4. sudo vgchange -ay ceph-4152b049-a23c-42ae-9e75-1a50ae887905
	
	5. sudo udevadm control --reload-rules
	
	   5.1 sudo udevadm trigger
	
	6. sudo ceph-volume lvm activate 4 83e3fce6-c029-41e0-9f1c-ff7de3aba9e3
	
	7. systemctl start ceph-osd@4.service

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------
My case tested through those steps and result:


	root@prx-lab:/var/lib/ceph/osd/ceph-4# systemctl stop ceph-osd@4.service
	root@prx-lab:/var/lib/ceph/osd/ceph-4# sudo lvs -o lv_name,vg_name,lv_uuid
	  LV                                             VG                                        LV UUID                               
	  osd-block-83e3fce6-c029-41e0-9f1c-ff7de3aba9e3 ceph-4152b049-a23c-42ae-9e75-1a50ae887905 llrRwQ-HuQ8-GFAf-mlpu-R1I2-EpNf-d9VKzn
	  osd-block-574fe4a3-5cb9-4564-b619-746dc664389c ceph-e7473915-eb5f-4dc5-bfb2-33a93045c48f Q8Pj8Y-M5pw-FDOB-Y9vt-tKcu-IHvb-K0dkcG
	  data                                           pve                                       rskBPI-kckN-fqGE-xu2e-J2fq-ml6m-8KyaHl
	  root                                           pve                                       KNRJIK-dywb-k0fH-w9lF-NO4q-PVCg-EC9Bpw
	  swap                                           pve                                       HrRRS8-vW5w-l5Ff-tc1c-ahD3-7aIF-EARXQx



	change the slot here



	root@prx-lab:/var/lib/ceph/osd/ceph-4# sudo vgchange -an ceph-4152b049-a23c-42ae-9e75-1a50ae887905
	  0 logical volume(s) in volume group "ceph-4152b049-a23c-42ae-9e75-1a50ae887905" now active
	  
	  
	  
	root@prx-lab:/var/lib/ceph/osd/ceph-4# sudo vgchange -ay ceph-4152b049-a23c-42ae-9e75-1a50ae887905
	  1 logical volume(s) in volume group "ceph-4152b049-a23c-42ae-9e75-1a50ae887905" now active
	  
	   
	root@prx-lab:/var/lib/ceph/osd/ceph-4# sudo udevadm control --reload-rules
	sudo udevadm trigger


	root@prx-lab:/var/lib/ceph/osd/ceph-4# sudo ceph-volume lvm activate 4 83e3fce6-c029-41e0-9f1c-ff7de3aba9e3
	Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-4
	Running command: /usr/bin/ceph-bluestore-tool --cluster=ceph prime-osd-dir --dev /dev/ceph-4152b049-a23c-42ae-9e75-1a50ae887905/osd-block-83e3fce6-c029-41e0-9f1c-ff7de3aba9e3 --path /var/lib/ceph/osd/ceph-4 --no-mon-config
	Running command: /usr/bin/ln -snf /dev/ceph-4152b049-a23c-42ae-9e75-1a50ae887905/osd-block-83e3fce6-c029-41e0-9f1c-ff7de3aba9e3 /var/lib/ceph/osd/ceph-4/block
	Running command: /usr/bin/chown -h ceph:ceph /var/lib/ceph/osd/ceph-4/block
	Running command: /usr/bin/chown -R ceph:ceph /dev/dm-1
	Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-4
	Running command: /usr/bin/systemctl enable ceph-volume@lvm-4-83e3fce6-c029-41e0-9f1c-ff7de3aba9e3
	Running command: /usr/bin/systemctl enable --runtime ceph-osd@4
	Running command: /usr/bin/systemctl start ceph-osd@4
	--> ceph-volume lvm activate successful for osd ID: 4




	root@prx-lab:/var/lib/ceph/osd/ceph-4# systemctl start ceph-osd@4.service
	root@prx-lab:/var/lib/ceph/osd/ceph-4# 
	root@prx-lab:/var/lib/ceph/osd/ceph-4# 
	root@prx-lab:/var/lib/ceph/osd/ceph-4# 

--------------------------------------------------------------------------------------------------------------------------------------------------------------



Here is the updated and corrected sequence of commands to reactivate LVM and start the OSD without rebooting. 
Step 1: Deactivate and re-activate the LVM volume group

    Stop the OSD: You have already done this correctly.
    sh

    systemctl stop ceph-osd@4.service


Deactivate the volume group: This ensures LVM has a clean state before re-scanning.
sh

sudo vgchange -an ceph-4152b049-a23c-42ae-9e75-1a50ae887905


Re-activate the volume group: This command will discover and re-establish the logical volumes, which is the key to solving the problem without a reboot.
sh

sudo vgchange -ay ceph-4152b049-a23c-42ae-9e75-1a50ae887905

 

Step 2: Trigger a udev rescan
Ensure that udev has the latest information about the device mapper. 
sh

sudo udevadm control --reload-rules
sudo udevadm trigger


Step 3: Use ceph-volume to activate the OSD
This is the most reliable method for ceph-volume managed OSDs, as it handles all the necessary Ceph-specific activation steps. 
sh

sudo ceph-volume lvm activate 4 83e3fce6-c029-41e0-9f1c-ff7de3aba9e3

Step 4: Start the OSD service
After activating the LVM volume, start the OSD service. 
sh

systemctl start ceph-osd@4.service

Future prevention
To avoid this transient LVM device discovery issue in the future, it is best to perform disk hot-swaps under controlled conditions. The safest procedure is to take the OSD offline using the steps above before moving the disk. After the move, run the manual LVM re-activation steps, and then bring the OSD back online. This is the recommended practice for live disk changes, even with LVM. 













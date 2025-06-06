# Recovering a disk array after one of the disks fails

This article tells you how to recover a disk in a RAID 1 array built on two [NRDs](https://yandex.cloud/docs/compute/concepts/disk#nr-disks).

This scenario was tested on the **Ubuntu 20.04 LTS** Linux distribution. 

See [Disk recovery](#dr) for the step by step guide.

## Prerequisites

* An existing Yandex Cloud folder.
* Installed [YC CLI](https://yandex.cloud/docs/cli/quickstart).
* Configured YC [profile](https://yandex.cloud/docs/cli/operations/authentication/service-account).

In this solution, the VM is deployed with a RAID 1 disk array built from two separate NRDs. If one of the disks in the array fails, you can recover it (i.e., replace with another).  

## Device IDs and names
Replacing a failed disk is a critical issue: misidentifying your target object can lead to deleting data in the wrong place. To avoid such situations, we recommend that you **always** access a disk through its `device-id` rather than use the device name, i.e., */dev/vdb*, for such operations. 

The `device-id` for your disk may look different inside the VM. If the **--device-name nrd2** key was specified when creating the disk via the YC CLI, this is the name that will be part of the `device-id` inside the VM, i.e., `virtio-nrd2` (see the output below). If no **--device-name** key was used when creating the disk, your VM's `device-id` will look different, i.e., `virtio-fhmjrhl1vjv8p2j68cm8` (see the output below).

While device IDs are usually unique, device names can change dynamically when devices are added and removed.

To see the relationship between `device-id` and `device-name`, you can use the `ls -l /dev/disk/by-id/` command. Below, you can see what it returns:

`$ ls -l /dev/disk/by-id/`
```bash
lrwxrwxrwx 1 root root 11 Oct 15 11:30 md-name-nrd-raid-test:100 -> ../../md100
lrwxrwxrwx 1 root root 11 Oct 15 11:30 md-uuid-c4ba6e99:0b18df98:5dc8a734:ac8a6d48 -> ../../md100
lrwxrwxrwx 1 root root  9 Oct 15 11:29 virtio-fhmjrhl1vjv8p2j68cm8 -> ../../vda
lrwxrwxrwx 1 root root 10 Oct 15 11:29 virtio-fhmjrhl1vjv8p2j68cm8-part1 -> ../../vda1
lrwxrwxrwx 1 root root 10 Oct 15 11:29 virtio-fhmjrhl1vjv8p2j68cm8-part2 -> ../../vda2
lrwxrwxrwx 1 root root  9 Oct 15 11:30 virtio-nrd1 -> ../../vdb
lrwxrwxrwx 1 root root  9 Oct 15 11:30 virtio-nrd2 -> ../../vdc 
```

## Tips for creating non-replicated disks (NRDs) in Yandex Cloud
* [Non-replicated SSD](https://yandex.cloud/docs/compute/concepts/disk) (`network-ssd-nonreplicated`): Network drive with enhanced performance.
* When creating an NRD via Terraform or the YC CLI, always set the `device-name` parameter to specify a disk name that you can easily recognize.
* To increase the fault tolerance of the disk subsystem, you may want to use [placement groups](https://yandex.cloud/docs/compute/concepts/disk-placement-group) for NRDs to place disks in different racks in the data center.

## Deploying the VM and emulating a disk failure

### Deployment plan
  1. [Create a disk placement group](https://yandex.cloud/docs/compute/operations/disk-placement-groups/create)
  2. [Create the non-replicated disks in the placement group](https://yandex.cloud/docs/compute/operations/disk-create/nonreplicated#nr-disk-in-group)
  3. [Create the VM and attach the created NRDs to it](https://yandex.cloud/docs/compute/operations/vm-create/create-linux-vm)

Feel free to use the Terraform code below as an example to deploy your VM.

### Deploying a VM using Terraform

#### Set the deployment parameters in [**variables.tf**](./variables.tf):
* `vm_name`: VM name.
* `vm_zone`: Name of the availability zone to deploy your VM in.
* `vm_disk_size`: Size of one disk, in GB (specify in multiples of 93 GB). All disks will be created with the same size.
* `vm_image`: Image to deploy your VM from.

#### Run the deployment
1. In [**env-yc-prod.sh**](./env-yc-prod.sh), check if the YC runtime environment is properly configured.
   Replace the `prod` profile name with that of the one you need.

2. Run the runtime environment:
    `source env-yc-prod.sh`

3. Initialize Terraform:
    `terraform init`

4. Deploy your VM:
    `terraform apply`

5. Connect to the VM over SSH:
    `ssh admin@<vm-public-ip>`

6. Check the health of your disks after deployment:
    `$ ls -l /dev/disk/by-id/`
    ```bash
    md-name-nrd-raid-test:100                   -> ../../md100
    md-uuid-663a720f:b0c8ab74:be2d83ae:70677d6d -> ../../md100
    virtio-fhmni7jprp8264t8i5mg                 -> ../../vda
    virtio-fhmni7jprp8264t8i5mg-part1           -> ../../vda1
    virtio-fhmni7jprp8264t8i5mg-part2           -> ../../vda2
    virtio-nrd1                                 -> ../../vdb
    virtio-nrd2                                 -> ../../vdc
    ```

    `$ sudo mdadm --query /dev/md100`
      ```bash
      /dev/md100: 92.94GiB raid1 2 devices, 0 spares. Use mdadm --detail for more detail.
      ```

    `$ sudo mdadm --detail /dev/md100`
      ```bash
      /dev/md100:
                Version  : 1.2
          Creation Time  : Sun Oct 10 16:14:26 2021
              Raid Level : raid1
              Array Size : 97451008 (92.94 GiB 99.79 GB)
          Used Dev Size  : 97451008 (92.94 GiB 99.79 GB)
            Raid Devices : 2
          Total Devices  : 2
            Persistence  : Superblock is persistent

            Update Time  : Sun Oct 10 16:30:37 2021
                  State  : clean
          Active Devices : 2
        Working Devices  : 2
          Failed Devices : 0
          Spare Devices  : 0

      Consistency Policy : resync

                    Name : nrd-raid-test:100 (local to host nrd-raid-test)
                    UUID : 663a720f:b0c8ab74:be2d83ae:70677d6d
                  Events : 29

          Number   Major   Minor   RaidDevice State
            0     252       16        0      active sync   /dev/vdb
            1     252       32        1      active sync   /dev/vdc
      ```

7. Write test data to a disk:
```bash
$ sudo -i
$ echo "test-1234567890-987654321-abcdefghiklmnopqrstuvwxyz" > /data/test.txt
$ cat /data/test.txt
$ sha1sum /data/test.txt # -> 1108f5f9021d4ba066da47debaa24aea28d8bc9b
```

### Emulating that one of the disks in your RAID array fails
In our example, there are two disks combined into a single [RAID 1](https://ru.wikipedia.org/wiki/RAID#RAID_1) array; such an array ensures that your data will be all safe if one of the disks fails. To quickly regain the array fault tolerance level in this case, replacing the failed disk is crucial.

For prompt notifications of such problems, you may want to set up [Incident notification](https://console.yandex.cloud/cloud?section=incident-notifications): with this feature, you can see who is already on your cloud's incident alert list, and adjust the list as you need. For more information, see the [relevant guide](https://yandex.cloud/docs/resource-manager/operations/cloud/notify).

In practice, you need to understand the condition of your disk before performing any operations on it. A healthy disk should report `READY` in the `status` field, while a failed one's status will be `ERROR`. You can check the disk status using the YC CLI command, specifying the disk ID as an argument, e.g., `yc compute disk get fhmd0e91sfmksudo9io4`. The result will be as follows:

```yml
id: fhmd0e91sfmksudo9io4
folder_id: u1g32j0789iog4d3cnk3
created_at: "2021-10-16T11:53:12Z"
name: nrd1
type_id: network-ssd-nonreplicated
zone_id: ru-central1-a
size: "99857989632"
block_size: "4096"
status: READY
disk_placement_policy:
  placement_group_id: fhm26q5eq2341rgbmhgs
```

`$ sudo mdadm --manage /dev/md100 --fail /dev/disk/by-id/virtio-nrd1`
```bash
mdadm: set /dev/disk/by-id/virtio-nrd1 faulty in /dev/md100
```

## Recovering the disk <a id="dr"/>

  1. [Check the health of your disk array](#dr-1)
  2. [Check the failed disk's `device-id`](#dr-2)
  3. [Remove the failed disk from the array](#dr-3)
  4. [Retrieve a list of disk placement groups](#dr-4)
  5. [Identify the failed disk's placement group](#dr-5)
  6. [Detach the failed disk from the VM](#dr-6)
  7. [Create a new NRD](#dr-7)
  8. [Attach the new disk to the VM](#dr-8)
  9. [Check if the new disk is available after attaching](#dr-9)
  10. [Copy the partition table from the working disk to the new one](#dr-10)
  11. [Add the new disk to the RAID array](#dr-11)
  12. [Check the array health after adding the disk](#dr-12)
  13. Check the data integrity

### 1. Check the disk array's health <a id="dr-1"/>

```bash
/dev/md100:
           Version : 1.2
     Creation Time : Mon Oct 11 15:50:12 2021
        Raid Level : raid1
        Array Size : 97451008 (92.94 GiB 99.79 GB)
     Used Dev Size : 97451008 (92.94 GiB 99.79 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Tue Oct 12 09:18:37 2021
             State : clean, degraded 
    Active Devices : 1
   Working Devices : 1
    Failed Devices : 1
     Spare Devices : 0

Consistency Policy : resync

              Name : nrd-raid-test:100  (local to host nrd-raid-test)
              UUID : b64eb7cd:6c0c1924:872fe4c0:bb5487c8
            Events : 23

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1     252       32        1      active sync   /dev/vdc
       0     252       16        -      faulty        /dev/vdb
```

`$ sudo cat /proc/mdstat`
```bash
Personalities : [raid1] 
md100 : active raid1 vdc[1] vdb[0](F)
      97451008 blocks super 1.2 [2/1] [_U]      
unused devices: <none>
```
The failed disk will be marked with `(F)`.


### 2. Check the failed disk's `device-id` <a id="dr-2"/>

  `$ ls -l /dev/disk/by-id/`

    ```bash
    md-name-nrd-raid-test:100 -> ../../md100
    md-uuid-9a5e8799:5d6db030:e7c959c2:38ce99ce -> ../../md100
    virtio-fhm8rsgmrjc9vmi1kj85 -> ../../vda
    virtio-fhm8rsgmrjc9vmi1kj85-part1 -> ../../vda1
    virtio-fhm8rsgmrjc9vmi1kj85-part2 -> ../../vda2
    virtio-nrd1 -> ../../vdb
    virtio-nrd2 -> ../../vdc
    ```

### 3. Remove the failed disk from the array <a id="dr-3"/> 
Remove the failed disk from the array by `device-id`:

`$ sudo mdadm --manage /dev/md100 --remove /dev/disk/by-id/virtio-nrd1`
```bash
mdadm: hot removed /dev/disk/by-id/virtio-nrd1 from /dev/md100
```

### 4. Retrieve a list of disk placement groups <a id="dr-4"/>

`yc compute disk-placement-group list`
```bash
+----------------------+------+---------------+--------+
|          ID          | NAME |     ZONE      | STATUS |
+----------------------+------+---------------+--------+
| fhm26q5eq2341rgbmhgs |      | ru-central1-a | READY  |
+----------------------+------+---------------+--------+
```

### 5. Identify the failed disk's placement group <a id="dr-5"/> 
Check the disks in each placement group until you identify the failed disk. Keep the placement group ID.

`yc compute disk-placement-group list-disks --id fhm26q5eq2341rgbmhgs`
```bash
+----------------------+------+-------------+---------------+--------+----------------------+-------------+
|          ID          | NAME |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | DESCRIPTION |
+----------------------+------+-------------+---------------+--------+----------------------+-------------+
| fhmd0e91sfmksudo9io4 | nrd1 | 99857989632 | ru-central1-a | READY  | fhmtfc9ggo2qc6ude23e |             |
| fhmidi40rdmj4km5u9vd | nrd2 | 99857989632 | ru-central1-a | READY  | fhmtfc9ggo2qc6ude23e |             |
+----------------------+------+-------------+---------------+--------+----------------------+-------------+
```

### 6. Detach the failed disk from the VM <a id="dr-6"/>

`yc compute instance list`
```bash
+----------------------+---------------+---------------+---------+-----------------+-------------+
|          ID          |     NAME      |    ZONE ID    | STATUS  |   EXTERNAL IP   | INTERNAL IP |
+----------------------+---------------+---------------+---------+-----------------+-------------+
| fhmtfc9ggo2qc6ude23e | nrd-raid-test | ru-central1-a | RUNNING | 62.84.119.106   | 10.128.0.29 |
+----------------------+---------------+---------------+---------+-----------------+-------------+
```
`yc compute instance detach-disk nrd-raid-test --disk-id fhmd0e91sfmksudo9io4`
```yml
done (11s)
id: fhmtfc9ggo2qc6ude23e
folder_id: b1g44j0674iog4o3ovh7
created_at: "2021-10-16T11:53:17Z"
name: nrd-raid-test
zone_id: ru-central1-a
platform_id: standard-v1
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
boot_disk:
  mode: READ_WRITE
  device_name: fhmvcmo4p8gs8vu99k0r
  auto_delete: true
  disk_id: fhmvcmo4p8gs8vu99k0r
secondary_disks:
- mode: READ_WRITE
  device_name: nrd2
  disk_id: fhmidi40rdmj4km5u9vd
network_interfaces:
- index: "0"
  mac_address: d0:0d:1d:7b:13:08
  subnet_id: e9bkhb79tegftff4tfqf
  primary_v4_address:
    address: 10.128.0.29
    one_to_one_nat:
      address: 62.84.119.106
      ip_version: IPV4
fqdn: nrd-raid-test.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```

### 7. Create a new NRD <a id="dr-7"/> 
Create a new NRD (`nrd3`) and place it to the existing placement group:

`yc compute disk create nrd3 \`\
`--type network-ssd-nonreplicated \`\
`--size 93 --zone ru-central1-a \`\
`--disk-placement-group-id fhm26q5eq2341rgbmhgs`
```yml
done (5s)
id: fhmlnioblb9co9pill3e
folder_id: b1g44j0674iog4o3ovh7
created_at: "2021-10-17T07:04:38Z"
name: nrd3
type_id: network-ssd-nonreplicated
zone_id: ru-central1-a
size: "99857989632"
block_size: "4096"
status: READY
disk_placement_policy:
  placement_group_id: fhm26q5eq2341rgbmhgs
```

### 8. Attach the new disk to the VM <a id="dr-8"/> 

`yc compute instance attach-disk --name nrd-raid-test --disk-name nrd3 --device-name nrd3 --mode rw`
```yml
done (3s)
id: fhmtfc9ggo2qc6ude23e
folder_id: b1g44j0674iog4o3ovh7
created_at: "2021-10-16T11:53:17Z"
name: nrd-raid-test
zone_id: ru-central1-a
platform_id: standard-v1
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
boot_disk:
  mode: READ_WRITE
  device_name: fhmvcmo4p8gs8vu99k0r
  auto_delete: true
  disk_id: fhmvcmo4p8gs8vu99k0r
secondary_disks:
- mode: READ_WRITE
  device_name: nrd2
  disk_id: fhmidi40rdmj4km5u9vd
- mode: READ_WRITE
  device_name: nrd3
  disk_id: fhmlnioblb9co9pill3e
network_interfaces:
- index: "0"
  mac_address: d0:0d:1d:7b:13:08
  subnet_id: e9bkhb79tegftff4tfqf
  primary_v4_address:
    address: 10.128.0.29
    one_to_one_nat:
      address: 62.84.119.106
      ip_version: IPV4
fqdn: nrd-raid-test.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```

### 9. Check if the new disk is available after attaching <a id="dr-9"/>

`$ ls -l /dev/disk/by-id/`

  ```bash
  md-name-nrd-raid-test:100 -> ../../md100
  md-uuid-9a5e8799:5d6db030:e7c959c2:38ce99ce -> ../../md100
  virtio-fhm8rsgmrjc9vmi1kj85 -> ../../vda
  virtio-fhm8rsgmrjc9vmi1kj85-part1 -> ../../vda1
  virtio-fhm8rsgmrjc9vmi1kj85-part2 -> ../../vda2
  virtio-nrd2 -> ../../vdc
  virtio-nrd3 -> ../../vdb
  ```

### 10. Copy the partition table from the working disk to the new one <a id="dr-10"/>

`$ sudo sfdisk -d /dev/disk/by-id/virtio-nrd2 | sudo sfdisk /dev/disk/by-id/virtio-nrd3`
```bash
Checking that no-one is using this disk right now ... OK

Disk /dev/disk/by-id/virtio-nrd3: 93 GiB, 99857989632 bytes, 195035136 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Old situation:

Device                            Boot Start       End   Sectors Size Id Type
/dev/disk/by-id/virtio-nrd3-part1          1 195035135 195035135  93G ee GPT

Partition 1 does not start on physical sector boundary.

>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Created a new DOS disklabel with disk identifier 0x00000000.
/dev/disk/by-id/virtio-nrd3-part1: Created a new partition 1 of type 'GPT' and of size 93 GiB.
/dev/disk/by-id/virtio-nrd3-part2: Done.

New situation:
Disklabel type: dos
Disk identifier: 0x00000000

Device                            Boot Start       End   Sectors Size Id Type
/dev/disk/by-id/virtio-nrd3-part1          1 195035135 195035135  93G ee GPT

Partition 1 does not start on physical sector boundary.

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

### 11. Add the new disk to the RAID array <a id="dr-11"/>

`$ sudo mdadm --manage /dev/md100 --add /dev/disk/by-id/virtio-nrd3`
```bash
mdadm: added /dev/disk/by-id/virtio-nrd3
```

### 12. Check the array health after adding the disk <a id="dr-12"/>

`$ sudo mdadm --detail /dev/md100`
```bash
/dev/md100:
           Version : 1.2
     Creation Time : Sat Oct 16 11:54:57 2021
        Raid Level : raid1
        Array Size : 97451008 (92.94 GiB 99.79 GB)
     Used Dev Size : 97451008 (92.94 GiB 99.79 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Sun Oct 17 07:12:14 2021
             State : clean, degraded, recovering
    Active Devices : 1
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 1

Consistency Policy : resync

    Rebuild Status : 11% complete

              Name : nrd-raid-test:100  (local to host nrd-raid-test)
              UUID : 95f26a09:a7e89ebf:65dac840:815cb020
            Events : 27

    Number   Major   Minor   RaidDevice State
       2     252       16        0      spare rebuilding   /dev/vdb
       1     252       32        1      active sync   /dev/vdc
```

`$ cat /proc/mdstat`
```bash
Personalities : [raid1]
md100 : active raid1 vdb[2] vdc[1]
      97451008 blocks super 1.2 [2/1] [_U]
      [===>.................]  recovery = 15.8% (15470464/97451008) finish=13.4min speed=101943K/sec

unused devices: <none>
```

You can see in the command output above that the rebuild/recovery process of the disk array is in progress.
Once the recovery is complete, the output should be as follows:

`$ sudo mdadm --detail /dev/md100`
```bash
/dev/md100:
           Version : 1.2
     Creation Time : Sat Oct 16 11:54:57 2021
        Raid Level : raid1
        Array Size : 97451008 (92.94 GiB 99.79 GB)
     Used Dev Size : 97451008 (92.94 GiB 99.79 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Sun Oct 17 07:27:16 2021
             State : clean
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : nrd-raid-test:100  (local to host nrd-raid-test)
              UUID : 95f26a09:a7e89ebf:65dac840:815cb020
            Events : 43

    Number   Major   Minor   RaidDevice State
       2     252       16        0      active sync   /dev/vdb
       1     252       32        1      active sync   /dev/vdc
```

`$ cat /proc/mdstat`
```bash
Personalities : [raid1]
md100 : active raid1 vdb[2] vdc[1]
      97451008 blocks super 1.2 [2/2] [UU]

unused devices: <none>
```

### 13. Check the data integrity


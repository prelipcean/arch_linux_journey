The key difference between `gdisk` and `fdisk` lies in the type of partition tables they manage:

- **fdisk:**
    - Works with Master Boot Record (MBR) partition tables.  
        
    - MBR is an older partitioning scheme with limitations, such as a maximum of four primary partitions and a 2TB disk size limit.  
        
- **gdisk:**
    - Works with GUID Partition Table (GPT) partition tables.  
        
    - GPT is a newer, more flexible partitioning scheme that overcomes the limitations of MBR. It supports a much larger number of partitions and significantly larger disk sizes.  
        

Here's a breakdown:

- **Partition Table Types:**
    - **MBR (Master Boot Record):**
        - Older standard.
        - Limitations in partition number and disk size.  
            
        - `fdisk` is used to manipulate MBR partition tables.
    - **GPT (GUID Partition Table):**
        - Newer standard.
        - More flexible and supports larger disks and more partitions.  
            
        - `gdisk` is used to manipulate GPT partition tables.
- **When to Use Which:**
    - Use `fdisk` when working with disks that use MBR partition tables.
    - Use `gdisk` when working with disks that use GPT partition tables, which is increasingly common, especially with larger hard drives and modern systems using UEFI.
- **UEFI vs. BIOS:**
    - GPT is typically associated with UEFI (Unified Extensible Firmware Interface) systems, while MBR is associated with older BIOS (Basic Input/Output System) systems.  
        

In essence, the choice between `gdisk` and `fdisk` depends on the partition table type of the disk you're working with.
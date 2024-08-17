# Flashing the PineNote using rkdeveloptool

This approach, if successfull, allows you to flash a new operating system to the PineNote without the need for the UART adapter. A working usb-c cable and the pen (i.e., a magnet) should suffice.

**NOTE:** In case something goes wrong the UART adapter could be required. If things go really wrong, opening up the device could also be required!

## Workflow for repartitioning and flashing
* See troubleshooting sections below for solutions to common problems
* Refer to the next section for information on the standard partition layout that is promoted here
* Make sure to:
	* Have a working installation of Pine64's fork of
	  [rkdeveloptool](https://gitlab.com/pine64-org/quartz-bsp/rkdeveloptool).
	  Note as of 6. July 2023, this fork is in Debian unstable (check for version number beginning with 1.32+pine64).
	* Have a full backup of the PineNote
	* Be ready to recover from any errors (i.e., have the UART-board ready or
	  tools to open up the PineNote)
	* You need to have a u-boot that can access data beyond 32 mb on the disc and that automatically detects extlinux.conf files on the partitions.
	  There are (at least) two possible ways to get such a u-boot:
	    * Patch the factory-flashed u-boot:
	      * See https://github.com/DorianRudolph/pinenotes#fix-uboot for fixing the 32mb problem (there is a link to a backup binary that you could use)
	      * Enabling the "search-for-exlinux.conf-file"-functionality in uboot can be accomplished by modifying the environment of the modified uboot partition (e.g., the backup provided by DorianRudolp). Use the file pinenote-uboot-envtool.py from  https://gist.github.com/charasyn/206b2537534b6679b0961be64cf9c35f, but instead of using the u-boot-patch provided by charasyn, just replace the *bootcmd* command of the environment with `bootcmd=run distro_bootcmd;` This modified image can be flashed using `rkdeveloptool write-partition uboot modifie_uboot.img
	    * **(preferred option)** An alternative u-boot (idblock.bin, uboot.img) can be found in the **uboot files**-artifact of the CI builds of this repository. Note that this u-boot version does only boot extlinux.conf linux distributions by default - you will loose (easy) access to any android systems.
	* For writing the partition table, you need the `rk356x_spl_loader_v1.12.112.bin` file (there are newer ones available, but this version has been verified to work. See troubleshooting section at the end of this document). This file can either be directly created from the rkbin repository, or should be available as a pre-built artifact in the latest CI build or in the latest release.
 	To generate it from the rockchip-provided binaries, use the following commands::

            git clone --shallow-since="2022-01-02T00:00:00Z" https://github.com/rockchip-linux/rkbin
            cd rkbin
            git checkout b6354b9
            tools/boot_merger RKBOOT/RK3566MINIALL.ini

* Preparation:
        * Download the new partition table file (from this repository): [partition_table_standard1.txt](partition_table_standard1.txt)
	* From the latest release (or latest CI build), download the following artifacts:
         * the spl loader:
           * **rk356x_spl_loader_v1.12.112.bin**
         * (optional): the u-boot artifacts:
           * **idblock.bin**
           * **uboot.img**
         * (optional): the logo artifact:
           * **logo.img**
	* unzip the artifacts

* Flashing commands:

	  # create backups (do this BEFORE altering any of the partitions!)
	  # If you already altered partitions, skip this step and write back
	  # partition backup you did before in the later step
	  rkdeveloptool read 0 41943040 first_40mb_of_disc.img
	  rkdeveloptool read-partition boot part_boot.img
	  rkdeveloptool read-partition trust part_trust.img
	  rkdeveloptool read-partition dtbo part_dtbo.img
	  rkdeveloptool read-partition waveform part_waveform.img
	  rkdeveloptool read-partition uboot part_uboot.img
	  rkdeveloptool read-partition logo part_logo.img
	  rkdeveloptool read-partition recovery part_recovery.img

	  # enable `write-partition-table` commands
	  rkdeveloptool reboot-maskrom
	  rkdeveloptool boot rk356x_spl_loader_v1.12.112.bin

	  # write new GPT partition table
	  rkdeveloptool write-partition-table partition_table_standard1.txt

	  # (optional) write new u-boot
	  # idblock.bin is only required if you compiled u-boot yourself (the rockchip u-boot)
	  rkdeveloptool write 64 idblock.bin
	  rkdeveloptool write-partition uboot uboot.img

	  # write partitions that were moved
	  rkdeveloptool write-partition logo logo_new.img
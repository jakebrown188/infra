# HDD Burn-In Tests

## Description
After buying a new drive, I put it through a burn-in test to ensure the drive is fully working and will not fail early in the bathtub curve. Since this test is very strenuous to the drive, it will weed out drives that fail early.

## Known Limitations
The maximum number of blocks `badblocks` can scan at once is 4,294,967,296 (a 32-bit int). This may become a problem when scanning large drives with their native block size. For example, a 18TB HDD with a block size of 4096 has a total of 4,394,582,016 blocks (18000207937536 bytes / 4096 block size). Trying to use `badblocks` to scan 4,394,582,016 blocks would result in this error:
```
badblocks: Value too large for defined data type invalid end block (4394582016): must be 32-bit value
```
*To play around with the number of blocks generated by different block sizes, you can use [this spreadsheet](./Calculate%20Number%20of%20Blocks.xlsx).*

According to [this](https://superuser.com/a/681823) StackExchange post (and some other resources I've found online), if the block size (`-b`) in `badblocks` is set to a value larger than the drive's block size, the integrity of `badblocks` can be compromised (i.e. you can get false-negatives: no bad blocks found when they may still exist). The burn-in script I am using defaults to a 8192 block size which is larger than my drive's block size of 4096. For now, I am simply just accepting the risk. I am assuming that the script being referenced utilizes enough passes and other testing mechanisms to mitigate the risk. An update to the script splitting up `badblocks` passes to only operate on a portion of the drive at a time (i.e. run `badblocks` on the first half of the drive and then the second half) would mitigate this issue.

## Burn-In Procedure
1. Open up a tmux session
2. Clone Spearfoot's [disk burn-in script repo](https://github.com/Spearfoot/disk-burnin-and-testing)
   - Archive [here](./archive/) if need be
3. In a separate tmux pane, open up the following monitoring tools:
   - General system monitoring: `htop`
   - Drive temperature monitoring: `watch -n 1 sudo hddtemp -q /dev/sda`
4. In another separate tmux pane, open information about each drive being tested. **This is critical to ensure the wrong drive doesn't accidentally get wiped.** Make sure to match this information up when starting the disk burn-in script
    ```
    sudo fdisk -l | grep -E "/dev/sda|/dev/sdb" -A 4
    sudo hdparm -I /dev/sd[a,b] | grep -E "/dev/sda|/dev/sdb" -A 4
    ```
5. Start the disk burn-in script for each drive being tested. **This will completely wipe any data on the drive**
    ```
    sudo ./disk-burnin.sh -f /dev/sda
    ```

## Miscellaneous Helpful Commands
- Get the block size of a drive
  ```
  sudo hdparm -I /dev/sda | grep -i physical
  ```
- Get the total number of bytes on a drive
  ```
  sudo fdisk -l | grep /dev/sda
  ```
- `badblocks` command that Spearfoot's script ends up using by default
  ```
  badblocks -b 8192 -wsv -c 64 -e 1 -o burnin-${DISK_MODEL}_${SERIAL_NUMBER}.log ${DRIVE_PATH}
  ```

## Resources
- Perfect Media Server [New Drive Burn-In Rituals](https://perfectmediaserver.com/06-hardware/new-drive-burnin/)
  - My burn-in procedure was taken from here
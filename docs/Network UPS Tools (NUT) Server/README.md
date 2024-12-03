# Network UPS Tools (NUT) Server

## Description
NUT is used to monitor and configure a UPS. Other systems can query this server to determine when they should shut down gracefully before the UPS runs out of battery in a power outage. While the NUT project does provide a frontend, I've decided to go with [nut_webgui](https://github.com/SuperioOne/nut_webgui) instead. `nut_webgui` has a cleaner, more up-to-date UI and refreshes it's data automatically unlike the NUT project frontend.

## Deployment
Currently, I am planning to run this on a Ubuntu Server 24.04 VM with 1GB of RAM and 1 CPU core. After doing some testing, running both the NUT server and `nut_webgui` frontend only uses about 350M of RAM and less than 2% CPU. This is with `nut_webgui` refreshing it's data every second.

Below are the steps to deploy this server manually to a Ubuntu Server 24.04 VM using the CyberPower CP1500PFCLCD UPS.
1. Make a hard link pointing `libusb-1.0.so` to `libusb-1.0.so.0` with the below command. This is required because `nut_scanner` currently looks for `libusb-1.0.so` specifically when scanning for connected UPS units. Per [this issue](https://github.com/networkupstools/nut/issues/2431), there is currently an implemented fix for this merged into the repo but it will take some time for distribution packages to catch up.
    ```
    sudo ln /usr/lib/x86_64-linux-gnu/libusb-1.0.so.0 /usr/lib/x86_64-linux-gnu/libusb-1.0.so
    ```
2. Update the system and install the NUT packages
   ```
   sudo apt update
   sudo apt install nut nut-client nut-server
   ```
3. Get configuration information for your UPS
   ```
   sudo nut-scanner -U
   ```
   The scanner will provide output similar to this:
   ```
   [nutdev1]
        driver = "usbhid-ups"
        port = "auto"
        vendorid = "0764"
        productid = "0601"
        product = "CP1500PFCLCDa"
        serial = "CXXPU7016037"
        vendor = "CPS"
        bus = "002"
        device = "005"
        busport = "002"
        ###NOTMATCHED-YET###bcdDevice = "0200"
   ```
4. Make a backup of every default NUT configuration files and clear them out
   ```
   cd /etc/nut
   sudo su
   mkdir backup-configs
   for i in *; do cp $i backup-configs/$i.backup && > $i; done
   ```
5. Update `ups.conf`
   ```
   pollinterval = 1
   maxretry = 3
   [cyberpower]
        driver = "usbhid-ups"
        port = "auto"
        vendorid = "0764"
        productid = "0601"
        product = "CP1500PFCLCDa"
        serial = "CXXPU7016037"
        vendor = "CPS"
        bus = "002"
        device = "005"
        busport = "002"
        ###NOTMATCHED-YET###bcdDevice = "0200"
   ```
6. Update `upsmon.conf`
   ```
   MONITOR cyberpower@localhost 1 admin test_pass master
   ```
7. Update `upsd.conf`
   ```
   LISTEN 0.0.0.0 3493
   ```
8. Update `nut.conf`
   ```
   MODE=netserver
   ```
9. Update `upsd.users`
    ```
    [admin]
        password = test_pass
        actions = set
        actions = fsd
        instcmds = all
    ```
10. Reboot the VM or restart the services
    ```
    sudo reboot

    OR

    sudo systemctl restart nut-{server,client,monitor}
    sudo upsdrvctl stop
    sudo upsdrvctl start
    ```
11. Make a place for the `nut_webgui` data
    ```
    mkdir nut_webgui
    cd nut_webgui/
    ```
12. Create `docker-compose.yml` in the `nut_webgui` directory
    ```
    version: "3.3"
    services:
      nutweb:
        image: ghcr.io/superioone/nut_webgui:latest
        restart: always
        network_mode: host         # Share same host
        environment:
          POLL_FREQ: "1"
          POLL_INTERVAL: "1"
          UPSD_ADDR: "localhost"
          UPSD_PORT: "3493"
          UPSD_USER: "nut-admin"
          UPSD_PASS: "test_pass"
          PORT: "8080"               # Outgoing port
          LOG_LEVEL: "debug"
    ```
13. Start the `nut_webgui` container
    ```
    docker-compose up -d
    ```

## Resources
 - [Official NUT Documentation](https://networkupstools.org/documentation.html)
 - [Techno Tim NUT YouTube Video](https://www.youtube.com/watch?v=vyBP7wpN72c)
 - [Techno Tim NUT Writeup](https://technotim.live/posts/NUT-server-guide/)
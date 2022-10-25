# Getting started with AWS IoT FleetWise Edge Agent on Renesas R-Car S4

This section describes how to deploy AWS IoT FleetWise Edge Agent onto an Renesas [R-Car S4 Reference Board/Spider](https://www.renesas.com/jp/en/products/automotive-products/automotive-system-chips-socs/rtp8a779f0askb0sp2s-r-car-s4-reference-boardspider).

## Prerequisites

- **Renesas Electronics Corporation R-Car S4 Reference Board/Spider**
- **AWS IoT FleetWise Edge Agent Compiled for ARM64**
  — If you are using an EC2 Graviton instance as your development machine, you will have completed this already above.

  - _If you are using a local Intel x86_64 development machine running ubuntu 20.04_, you will need to run the following to cross-compile AWS IoT FleetWise Edge Agent:

    ```bash
    cd ~/aws-iot-fleetwise-edge
    CODENAME=$(grep UBUNTU_CODENAME /etc/os-release | cut -d'=' -f2 )
    # update the ubuntu version to your host
    sed -i "s/bionic/${CODENAME}/g" ./tools/arm64.list
    sudo -H ./tools/install-deps-cross.sh \
        && rm -rf build \
        && ./tools/build-fwe-cross.sh
    ```

## Build an SD-Card Image

The following instructions use the development machine(Ubuntu 20.04) to build an SD-card image based on the Ubuntu variant of the Renesas Linux BSP version 5.10.41.

1. Run the following _on the development machine_ to build SD card image:

   ```bash
    cd ~/aws-iot-fleetwise-edge \
       && sudo ./tools/renesas-rcar-s4/make-rootfs.sh 20.04.4 spider -sd
   ```

## Flash the SD-Card Image

1. Run the following to write the image to SD card:

   ```bash
    sudo dd if=./Ubuntu-20.04.4-rootfs-image-rcar-spider-sdcard.ext4 of=/dev/sdc bs=1M status=progress
   ```

## Specify Initial Board Configuration

1. Insert the SD-card into the R-Car S4 Spider board’s SD-card slot.
1. Connect an Ethernet cable.
1. Connect develop machine to R-Car S4 Spider board USB port.
1. Power ON S4 Spider board.
1. Use screen command on your develop machine terminal to veiw serial output.
1. Push Power button to boot the system. You can see the count down during U-Boot. Hit enter key to stop U-Boot.
1. Enter following settings to flash the SD card data to board

   ```bash
     setenv _booti 'booti 0x48080000 - 0x48000000'
     setenv sd1 'setenv bootargs rw root=/dev/mmcblk0p1 rootwait ip=dhcp maxcpus=1'
     setenv sd2 ext4load mmc 0:1 0x48080000 /boot/Image
     setenv sd3 ext4load mmc 0:1 0x48000000 /boot/r8a779f0-spider.dtb
     setenv sd 'run sd1; run sd2; run sd3; run _booti'
     setenv bootcmd 'run sd'
     saveenv
     boot
   ```

1. Connect to the R-Car S4 Spider board via SSH, entering password `rcar`:
   `ssh rcar@<R-Car Ip address>`
1. Once connected via SSH, check the board’s internet connection by running: `ping amazon.com`. There should be 0% packet loss.

## Provision AWS IoT Credentials

Run the following commands _on the development machine_ (after compiling AWS IoT FleetWise Edge Agent for ARM64 as explained above), to create an IoT Thing and provision credentials for it. The AWS IoT FleetWise Edge Agent binary and its configuration files will be packaged into a ZIP file ready to be deployed to the board.

```bash
mkdir -p ~/aws-iot-fleetwise-deploy && cd ~/aws-iot-fleetwise-deploy \
&& cp -r ~/aws-iot-fleetwise-edge/tools . \
&& mkdir -p build/src/executionmanagement \
&& cp ~/aws-iot-fleetwise-edge/build/src/executionmanagement/aws-iot-fleetwise-edge \
  build/src/executionmanagement/ \
&& mkdir -p config && cd config \
&& ../tools/provision.sh \
  --vehicle-name fwdemo-rcars4 \
  --certificate-pem-outfile certificate.pem \
  --private-key-outfile private-key.key \
  --endpoint-url-outfile endpoint.txt \
  --vehicle-name-outfile vehicle-name.txt \
&& ../tools/configure-fwe.sh \
  --input-config-file ~/aws-iot-fleetwise-edge/configuration/static-config.json \
  --output-config-file config-0.json \
  --vehicle-name `cat vehicle-name.txt` \
  --endpoint-url `cat endpoint.txt` \
  --can-bus0 vcan0 \
&& cd .. && zip -r aws-iot-fleetwise-deploy.zip .
```

## Deploy AWS IoT FleetWise Edge Agent software on R-Car S4 Spider board

1. Run the following _on your local machine_ to copy the deployment ZIP file from the EC2 machine to your local machine:

   ```bash
   scp -i <PATH_TO_PEM> ubuntu@<EC2_IP_ADDRESS>:aws-iot-fleetwise-deploy/aws-iot-fleetwise-deploy.zip .
   ```

1. Run the following _on your local machine_ to copy the deployment ZIP file from your local machine to the R-Car S4 Spider board (replace `<R-Car Ip address>` with the IP address of the R-Car S4 Spider board):

   ```bash
   scp aws-iot-fleetwise-deploy.zip rcar@<R-Car Ip address>:
   ```

1. SSH to the R-Car S4 Spider board, as described above, then run the following **_on the R-Car S4 Spider board_** to install AWS IoT FleetWise Edge Agent as a service:

   ```bash
    mkdir -p ~/aws-iot-fleetwise-deploy && cd ~/aws-iot-fleetwise-deploy \
       && unzip -o ~/aws-iot-fleetwise-deploy.zip \
       && sudo mkdir -p /etc/aws-iot-fleetwise \
       && sudo cp config/* /etc/aws-iot-fleetwise

    cp ./tools/install-socketcan.sh ./tools/setup-vcan.sh
    sed -i '17,68d' ./tools/setup-vcan.sh ./tools/setup-vcan.sh
    sudo ./tools/install-fwe.sh
    sed -i 's/python3.7/python3/g' ./tools/cansim/run-cansim.sh
    sed -i 's/python3.7/python3/g' ./tools/install-cansim.sh
    sed -i -e 's/^After/# After/' -e 's/^Wants/#Wants/' ./tools/cansim/cansim@.service
    sudo -H ./tools/install-cansim.sh
   ```

1. Run the following **_on the R-Car S4 Spider board_** to view and follow the AWS IoT FleetWise Edge Agent log (press CTRL+C to exit):

   ```bash
   sudo journalctl -fu fwe@0
   ```

## Collect OBD Data

1. Run the following _on the development machine_ to deploy a ‘heartbeat’ campaign that periodically collects OBD data:

   ```bash
   cd ~/aws-iot-fleetwise-edge/tools/cloud
   sed -i 's/python3.7/python3/g' ./install-deps.sh
   sudo -H ./install-deps.sh
   sed -i 's/python3.7/python3/g' ./demo.sh
   ./demo.sh --vehicle-name fwdemo-rcars4 --campaign-file campaign-obd-heartbeat.json
   ```
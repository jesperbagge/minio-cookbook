# Install a 4-node minio cluster on Ubuntu 18.04.4 LTS
This logbook was started 2020-04-23.

## Prerequisites

### Create user
Create a system user account to run minio as a systemd daemon

    sudo groupadd --system minio
    sudo useradd -s /sbin/nologin --system -g minio minio

### Create mounts / directories where minio stores data
In order to run Minio in fault tolerant mode (erasure code) a disk set of 4 is equired. If you plan on running a single node, these 4 disks should be mounted from 4 different disks. In this example, I'll be running one disk on 4 nodes each to achieve fault tolerance.

    sudo mkdir /data
    sudo chown -R minio:minio /data

## Download and install software
Download the software for 64bit Ubuntu

    wget https://dl.min.io/server/minio/release/linux-amd64/minio

The entire server software is one file. Make it executable

    chmod +x minio

Move it to a location where it can be run by a local system user that doesn't have a `/home` directory

    sudo mv minio /usr/local/bin

## Create an environment file for the `minio.service` file

TBD

## Create systemd service unit for minio

    sudo nano /etc/systemd/system/minio.service

Paste the following contents into `minio.service`

    [Unit]
    Description=minio
    Documentation=https://docs.minio.io
    Wants=network-online.target
    After=network-online.target
    AssertFileIsExecutable=/usr/local/bin/minio

    [Service]
    WorkingDirectory=/data
    User=minio
    Group=minio

    EnvironmentFile=-/etc/default/minio
    ExecStartPre=/bin/bash -c "if [ -z \"${minio_VOLUMES}\" ]; then echo \"Variable minio_VOLUMES not set in /etc/default/minio\"; exit 1; fi"

    ExecStart=/usr/local/bin/minio server $minio_OPTS $minio_VOLUMES

    # Let systemd restart this service always
    Restart=always

    # Specifies the maximum file descriptor number that can be opened by this process
    LimitNOFILE=65536

    # Disable timeout logic and wait until process is stopped
    TimeoutStopSec=infinity
    SendSIGKILL=no

    [Install]
    WantedBy=multi-user.target

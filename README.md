# Install a 4-node Minio cluster on Ubuntu 18.04.4 LTS
This logbook was started 2020-04-23.

## Prerequisites
TBD

## Download and install software
Download the software for 64bit Ubuntu

    wget https://dl.min.io/server/minio/release/linux-amd64/minio

The entire server software is one file. Make it executable

    chmod +x minio

Move it to a location where it can be run by a local system user that doesn't have a `/home` directory

    sudo mv minio /usr/local/bin

## Create user
Create a system user account to run MinIO as a systemd daemon

    sudo groupadd --system minio
    sudo useradd -s /sbin/nologin --system -g minio minio

## Create mounts / directories where MinIO stores data

    sudo mkdir /data
    sudo chown -R minio:minio /data

## Create an environment file for the `minio.service` file

TBD

## Create systemd service unit for MinIO

    sudo nano /etc/systemd/system/minio.service

Paste the following contents into `minio.service`

    [Unit]
    Description=Minio
    Documentation=https://docs.minio.io
    Wants=network-online.target
    After=network-online.target
    AssertFileIsExecutable=/usr/local/bin/minio

    [Service]
    WorkingDirectory=/data
    User=minio
    Group=minio

    EnvironmentFile=-/etc/default/minio
    ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"

    ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

    # Let systemd restart this service always
    Restart=always

    # Specifies the maximum file descriptor number that can be opened by this process
    LimitNOFILE=65536

    # Disable timeout logic and wait until process is stopped
    TimeoutStopSec=infinity
    SendSIGKILL=no

    [Install]
    WantedBy=multi-user.target

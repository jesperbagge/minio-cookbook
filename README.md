# Install a 4-node minio cluster on Ubuntu 18.04.4 LTS
The following steps needs to be performed on every node that should constitute the cluster.
Every node should have the same configuration file, ensuring that all nodes have the same access keys.

## Prerequisites
The preferred way (or atleast most convenient) to go about setting up a distributed Minio cluster is to name your nodes in a range-like manner, i.e. node1, node2, node3, node4, etc...
You will see why this is convenient later when we create our configuration file.

In this cookbook I have named my 4 nodes:
 * minio1
 * minio2
 * minio3
 * minio4

### Create user
Create a system user account to run minio as a systemd daemon

    sudo groupadd --system minio
    sudo useradd -s /sbin/nologin --system -g minio minio

### Create mounts / directories where minio stores data
In order to run Minio in fault tolerant mode (erasure code) a disk set of 4 is equired. If you plan on running a single node, these should be mounted from 4 different disks on the same node. In this example, I'll be running one disk on 4 nodes each to achieve fault tolerance.

    sudo mkdir -p /data/minio
    sudo chown -R minio:minio /data/minio

## Download and install software
Download the software for 64bit Ubuntu

    wget https://dl.min.io/server/minio/release/linux-amd64/minio

The entire server software is one file. Make it executable

    chmod +x minio

Move it to a location where it can be run by a local system user that doesn't have a `/home` directory

    sudo mv minio /usr/local/bin

## Create an environment file for the `minio.service` file
A file containing environent variable for minio should be placed at `/etc/default/minio`

    sudo mkdir /etd/default
    sudo nano /etc/default/minio

Paste the following contents into `/etc/default/minio`. Replace hostname with the name of your node(s). Remember the tip about naming your nodes in range?

    # Volume to be used for Minio server.
    MINIO_VOLUME="/data/minio"
    # Set range of nodenames.
    MINIO_OPTS="http://minio{1...4}"
    # Access Key of the server.
    MINIO_ACCESS_KEY=your-access-key-here
    # Secret key of the server.
    MINIO_SECRET_KEY=enter-your-secret-key-here

### Encryption
Auto-encryption can be turned on with the enviroment variable `MINIO_KMS_AUTO_ENCRYPTION`. For a production-grade set-up this would require a proper KMS system, like Hashicorp Vault. 
For lesser management overhead, Minio can be configured to use server-side encryption using a single master key. To set this up, begin with generating a random master key with the following command:

    head -c 32 /dev/urandom | xxd -c 32 -ps

The output of this command gives you a 256 bit key encoded as HEX. Use this with a master-key ID that you set yourself. Add the following lines to `/etc/default/minio`

    # Server-side encryption with single master key. Don't lose this! 
    MINIO_KMS_AUTO_ENCRYPTION=on
    MINIO_KMS_MASTER_KEY=your-key-id:a4080d8433737649d2e39f390aba7a7ef4e00256a73d52510ae5fb688a461c1d

Replace that long number with the output of your `head` command.

**Note! Don't generate different KMS master keys on each node! Remember that all objects will be encrypted at rest on the cluster - even if auto-encryption is turned off at a later time.** 


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
    WorkingDirectory=/data/minio
    User=minio
    Group=minio

    EnvironmentFile=/etc/default/minio

    ExecStart=/usr/local/bin/minio server $MINIO_OPTS$MINIO_VOLUME

    # Let systemd restart this service always
    Restart=always

    # Specifies the maximum file descriptor number that can be opened by this process
    LimitNOFILE=65536

    # Disable timeout logic and wait until process is stopped
    TimeoutStopSec=infinity
    SendSIGKILL=no

    [Install]
    WantedBy=multi-user.target

## Install the daemon and start 
Install the daemon

    sudo systemctl daemon-reload

Enable Minio start at boot

    sudo systemctl enable minio

Start Minio now

    sudo systemctl start minio


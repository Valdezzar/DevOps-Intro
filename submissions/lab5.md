# Lab 5 submission

## Task 1: Vagrant VM

I used Vagrant with VirtualBox to run QuickNotes inside an Ubuntu 24.04 VM.

The VM uses:

- Ubuntu 24.04
- 2 vCPUs
- 1024 MB RAM
- NAT networking
- host port `127.0.0.1:18080` forwarded to guest port `8080`
- VirtualBox shared folders
- Go 1.24.5
- shell provisioning

## Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.hostname = "quicknotes-lab5"

  config.vm.network "forwarded_port",
    guest: 8080,
    host: 18080,
    host_ip: "127.0.0.1",
    auto_correct: false

  config.vm.synced_folder "./app",
    "/opt/quicknotes/app",
    type: "virtualbox"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "quicknotes-lab5"
    vb.memory = 1024
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -eux

    GO_VERSION="1.24.5"

    apt-get update
    apt-get install -y curl ca-certificates

    CURRENT=""
    if [ -x /usr/local/go/bin/go ]; then
      CURRENT=$(/usr/local/go/bin/go version | awk '{print $3}')
    fi

    if [ "$CURRENT" != "go${GO_VERSION}" ]; then
      rm -rf /usr/local/go
      curl -fL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" -o /tmp/go.tar.gz
      tar -C /usr/local -xzf /tmp/go.tar.gz
      rm -f /tmp/go.tar.gz
    fi

    ln -sf /usr/local/go/bin/go /usr/local/bin/go
    ln -sf /usr/local/go/bin/gofmt /usr/local/bin/gofmt

    cd /opt/quicknotes/app
    /usr/local/go/bin/go build -o /usr/local/bin/quicknotes .

    printf '%s\n' \
      '[Unit]' \
      'Description=QuickNotes' \
      'After=network.target' \
      '' \
      '[Service]' \
      'Type=simple' \
      'WorkingDirectory=/opt/quicknotes/app' \
      'Environment=ADDR=:8080' \
      'ExecStart=/usr/local/bin/quicknotes' \
      'Restart=always' \
      'RestartSec=2' \
      '' \
      '[Install]' \
      'WantedBy=multi-user.target' \
      > /etc/systemd/system/quicknotes.service

    systemctl daemon-reload
    systemctl enable quicknotes
    systemctl restart quicknotes
  SHELL
end
```

## First `vagrant up`

The first lines of the initial run were:

```text
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'bento/ubuntu-24.04' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: >= 0
==> default: Loading metadata for box 'bento/ubuntu-24.04'
    default: URL: https://vagrantcloud.com/api/v2/vagrant/bento/ubuntu-24.04
==> default: Adding box 'bento/ubuntu-24.04' (v202510.26.0) for provider: virtualbox (amd64)
    default: Downloading: https://vagrantcloud.com/bento/boxes/ubuntu-24.04/versions/202510.26.0/providers/virtualbox/amd64/vagrant.box

[KProgress: 1% (Rate: 4245k/s, Estimated time remaining: 0:06:38)
```

The VM booted successfully.

```text
default running (virtualbox)
```

## Go version inside VM

Command:

```text
vagrant ssh -c "go version"
```

Output:

```text
go version go1.24.5 linux/amd64
```

## QuickNotes inside VM

Command:

```text
vagrant ssh -c "curl -s http://127.0.0.1:8080/health"
```

Output:

```json
{"notes":6,"status":"ok"}
```

## QuickNotes from Windows host

Command:

```text
curl http://127.0.0.1:18080/health
```

Output:

```json
{"notes":6,"status":"ok"}
```

The port forwarding from host port 18080 to guest port 8080 works.

## Design questions

### a) Synced folders

I used the VirtualBox shared-folder provider. It works directly with the VirtualBox provider and keeps the host `app` directory visible inside the VM without a separate sync command. The main trade-off is that shared-folder performance can be slower than a native Linux filesystem, especially with many small files.

### b) NAT vs Bridged vs Host-only

The VM uses NAT, which is Vagrant's default networking mode. The application port is forwarded only to `127.0.0.1` on the host. This is safer than a bridged adapter because the VM service is not directly exposed to other machines on the physical network.

### c) Provisioning

I used the shell provisioner. Installing one pinned Go version and creating one systemd service are simple tasks, so adding Ansible or another configuration-management tool would add unnecessary complexity for this lab.

### d) Why pin Go 1.24.5

Using exactly Go 1.24.5 makes provisioning reproducible. If the configuration only requested Go 1.24, a later point release could change compiler behavior or tooling without any change to the repository.

# Task 2: Snapshot, Break, Restore

## Save

I first verified that the VM was working and then created a snapshot:

```text
vagrant snapshot save clean-lab5
```

The snapshot list showed:

```text
clean-lab5
```

## Break

I deliberately removed the Go command and moved the Go installation:

```text
sudo rm -f /usr/local/bin/go /usr/local/bin/gofmt
sudo mv /usr/local/go /usr/local/go.broken
```

Verification:

```text
EXPECTED_FAILURE_GO_NOT_FOUND
```

This confirmed that the VM had been deliberately broken.

## Restore

I restored the VM with:

```text
vagrant snapshot restore clean-lab5 --no-provision
```

Restore time:

```text
21.39 seconds
```

## Verify recovery

After restoring the snapshot:

```text

```

The QuickNotes systemd service returned:

```text

```

Health check inside the VM:

```json

```

Health check from Windows through the forwarded port:

```json

```

The snapshot successfully restored the Go installation and the working QuickNotes service.

## Design questions

### e) Why snapshots are not backups

A snapshot depends on the original VM and its storage. If the host disk, VM files, or VirtualBox storage are lost or corrupted, the snapshot can disappear with them. A real backup should exist independently from the system it protects.

### f) Copy-on-write

A snapshot does not immediately create another full copy of the virtual disk. VirtualBox keeps the original state and stores later changes in additional differencing files. Ten snapshots therefore do not automatically use ten times the original disk size, but their changed blocks accumulate and can still consume a large amount of space.

### g) When snapshotting becomes an antipattern

Long snapshot chains make storage harder to manage and increase dependency on multiple differencing disks. They can also make recovery and deletion slower and more fragile. Snapshots are useful for short experiments and rollback points, but they should not replace reproducible provisioning or proper backups.

## Bonus

Not attempted.
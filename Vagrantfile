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
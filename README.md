Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation

Node Name

Your Node Name
Port prefix

273
# Download binary
cd && wget https://github.com/warden-protocol/wardenprotocol/releases/download/v0.5.2/wardend_Linux_x86_64.zip
unzip wardend_Linux_x86_64.zip
rm -rf wardend_Linux_x86_64.zip
chmod +x wardend
sudo mv wardend $HOME/go/bin/warndend

# Prepare cosmovisor directories
mkdir -p $HOME/.warden/cosmovisor/genesis/bin
ln -s $HOME/.warden/cosmovisor/genesis $HOME/.warden/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which wardend) $HOME/.warden/cosmovisor/genesis/bin

# Set node CLI configuration
wardend config set client chain-id chiado_10010-1
wardend config set client keyring-backend test
wardend config set client node tcp://localhost:27357

# Initialize the node
wardend init "Your Node Name" --chain-id chiado_10010-1

# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/warden/genesis.json > $HOME/.warden/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/warden/addrbook.json > $HOME/.warden/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "2d2c7af1c2d28408f437aef3d034087f40b85401@52.51.132.79:26656,2d2c7af1c2d28408f437aef3d034087f40b85401@52.51.132.79:26656,5461e7642520a1f8427ffaa57f9d39cf345fcd47@54.72.190.0:26656"|' $HOME/.warden/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "1award"|' $HOME/.warden/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.warden/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.warden/config/config.toml

# Change ports
sed -i -e "s%:1317%:27317%; s%:8080%:27380%; s%:9090%:27390%; s%:9091%:27391%; s%:8545%:27345%; s%:8546%:27346%; s%:6065%:27365%" $HOME/.warden/config/app.toml
sed -i -e "s%:26658%:27358%; s%:26657%:27357%; s%:6060%:27360%; s%:26656%:27356%; s%:26660%:27361%" $HOME/.warden/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/warden/warden_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.warden"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/warden.service > /dev/null << EOF
[Unit]
Description=Warden node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.warden
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.warden"
Environment="DAEMON_NAME=wardend"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable warden.service

# Start the service and check the logs
sudo systemctl start warden.service
sudo journalctl -u warden.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP

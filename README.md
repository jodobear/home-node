## Security
```bash
sudo ufw enable &&
sudo ufw allow ssh &&
sudo apt-get install openssh-server &&

# SSH config: disable root login, disable password auth
sudo sed -i 's/^PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config &&
# TODO: set nonstandard SSH port? instructions for setting up keys?

# Secure shared memory
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" | sudo tee -a /etc/fstab
```

Edit `/etc/sysctl.conf`, add:

```
# IP Spoofing protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignore ICMP broadcast requests
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Disable source packet routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0 
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Ignore send redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Block SYN attacks
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Log Martians
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0 
net.ipv6.conf.default.accept_redirects = 0

# Ignore Directed pings
net.ipv4.icmp_echo_ignore_all = 1
```

## Tor
```bash
sudo apt-get install tor
```

## Bitcoin Core

### Installing
```bash
# Create dir for installation files
mkdir ~/bitcoin-installation && cd ~/bitcoin-installation &&

# Download binaries
wget https://bitcoincore.org/bin/bitcoin-core-0.17.1/bitcoin-0.17.1-x86_64-linux-gnu.tar.gz &&
# Download signature
wget https://bitcoincore.org/bin/bitcoin-core-0.17.1/SHA256SUMS.asc &&

# Add signing key
gpg --recv-keys 01EA5486DE18A882D4C2684590C8019E36C2E964 &&
# Verify signature - should see XXXXXX
gpg --verify SHA256SUMS.asc &&
# Verify the downloaded binary matches the signed hash in SHA256SUMS.asc
grep bitcoin-0.17.1-x86_64-linux-gnu.tar.gz SHA256SUMS.asc | sha256sum -c - &&

# Unpack binaries
tar xvf bitcoin-0.17.1-x86_64-linux-gnu.tar.gz &&
# Install binaries system-wide (requires password)
sudo cp bitcoin-0.17.1/bin/* /usr/bin
```

### Configuring

Create `~/.bitcoin/bitcoin.conf`, add:

```bash
server=1
proxy=127.0.0.1:9050
disablewallet=1

# Check total memory with `free -m`, can be removed after initial sync
dbcache=???

# Optional (takes more space, needed for btc-rpc-explorer)
txindex=1
```

Generate random username/password for rpc access:

```bash
echo "
rpcuser=`head -c 5 /dev/urandom | base64 | tr -d '+/='`
rpcpassword=`head -c 30 /dev/urandom | base64 | tr -d '+/='`
" | tee -a ~/.bitcoin/bitcoin.conf
```

### Running
```bash
bitcoind
```

### Adding a startup service

TODO

## btc-rpc-explorer

### Installing
```bash
# Install dependencies
sudo apt install nodejs npm git &&
# Download source
git clone https://github.com/janoside/btc-rpc-explorer ~/btc-rpc-explorer &&
cd ~/btc-rpc-explorer &&
# Build source
npm install && npm run build
```

### Configuring
```bash
# Copy the bitcoind rpc user/pass from ~/.bitcoin/bitcoin.conf to ./app/credentials.js
sed -i -r 's/username:"[^"]+"/username:"'`grep ^rpcuser= ~/.bitcoin/bitcoin.conf | cut -d= -f2`'"/' app/credentials.js &&
sed -i -r 's/password:"[^"]+"/password:"'`grep ^rpcpassword= ~/.bitcoin/bitcoin.conf | cut -d= -f2`'""/' app/credentials.js
```

### Running
```bash
npm start
```

Then open http://localhost:3002/.

### Setup hidden service

TODO

### Adding a startup service

TODO

## Electrum Wallet

### Installing

```bash
# Install dependencies
sudo apt-get install python3-setuptools python3-pyqt5 python3-pip &&

# Create dir for installation files
mkdir ~/electrum-installation && cd ~/electrum-installation &&

# Download source
wget https://download.electrum.org/3.3.2/Electrum-3.3.2.tar.gz &&
# Download signature
wget https://download.electrum.org/3.3.2/Electrum-3.3.2.tar.gz.asc &&

# Add signing key
gpg --recv-keys 6694D8DE7BE8EE5631BE D9502BD5824B7F9470E6 &&
# Verify signature - should see XXX
gpg --verify Electrum-3.3.2.tar.gz.asc Electrum-3.3.2.tar.gz &&

# Unpack
tar xvf Electrum-3.3.2.tar.gz && cd Electrum-3.3.2 &&

# Install system-wide (requires sudo password)
sudo python3 -m pip install .[fast] &&
sudo python3 setup.py install
```

### Configuring
```
# Connect to local EPS server only
electrum setconfig server 127.0.0.1:50002:s &&
electrum setconfig oneserver true
```

### Running
Electrum can now be opened from the launcher or using the command line with `electrum`.

## Electrum Personal Server

### Installing
```
# Download source
git clone https://github.com/chris-belcher/electrum-personal-server.git ~/eps && cd ~/eps &&
# Checkout v0.1.6 (latest stable)
git checkout eps-v0.1.6 &&

# Add signing key
gpg --recv-keys 0A8B038F5E10CC2789BFCFFFEF734EA677F31129 &&
# Verify signature - should see XXX
git verify-commit HEAD &&

# Install system-wide (requires sudo password)
sudo pip3 install .
```

### Configuring

Copy sample configuration file with `cp ~/eps/config.cfg_sample ~/eps.cfg`.

Edit `~/eps.cfg`, add your xpubkey under `[master-public-keys]` as a new line with `{name}={xpubkey}`.

`{name}` can be anything, `{xpubkey}` should be copied from your Electrum wallet.

### Running
```bash
electrum-personal-server ~/eps.cfg
```

### Adding as a startup service

TODO

### Setup hidden service

TODO

## c-lightning

### Installing
```bash
# Install dependencies
sudo apt install autoconf automake build-essential libtool libgmp-dev libsqlite3-dev net-tools zlib1g-dev &&

# Download source
git clone https://github.com/ElementsProject/lightning ~/lightning && cd ~/lightning &&
# Checkout v0.6.3 (latest stable)
git checkout v0.6.3 &&

# Add signing key
gpg --recv-keys 15EE8D6CAB0E7F0CF999BFCBD9200E6CD1ADB8F1 &&
# Verify signature - should see XXX
git verify-tag v0.6.3 &&

# Build
./configure && make &&

# Install system-wide (requires sudo password)
sudo make install
```

### Adding as a startup service

TODO

## Spark Wallet

### Installing
```bash
# Install dependencies
sudo apt-get install nodejs npm &&

# Create dir for installation files
mkdir ~/spark-installation && cd ~/spark-installation &&

# Download npm package
wget https://github.com/shesek/spark-wallet/releases/download/v0.2.2/spark-wallet-0.2.2-npm.tgz &&
# Download signature
wget https://github.com/shesek/spark-wallet/releases/download/v0.2.2/SHA256SUMS.asc &&

# Add signing key
gpg --keyserver hkp://keyserver.ubuntu.com/ --recv-keys FCF19B67866562F08A43AAD681F6104CD0F150FC &&
# Verify signature - should show "Good signature from Nadav Ivgi <nadav@shesek.info>"
gpg --verify SHA256SUMS.asc &&
# Verify the downloaded binary matches the signed hash in SHA256SUMS.asc
grep spark-wallet-0.2.2-npm.tgz SHA256SUMS.asc | sha256sum -c - &&

# Install system-wide (requires sudo password)
sudo npm install -g spark-wallet-0.2.2-npm.tgz
```

### Configuring

Create `~/.spark-wallet/config`, set your username/password:

```
# this grants access to the funds in your lightning wallet, pick a strong password!
login={username}:{password}
```

Or generate random credentials with:

```bash
echo "login=`head -c 5 /dev/urandom | base64 | tr -d '+/='`:`head -c 30 /dev/urandom | base64 | tr -d '+/='`" | tee -a ~/.spark-wallet/config
```

### Running
```bash
spark-wallet
```

Then open http://localhost:9737/.

### Adding as a startup service

TODO

### Setup hidden service

TODO
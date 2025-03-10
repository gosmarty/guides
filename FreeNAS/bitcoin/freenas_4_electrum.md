[ [<< Back to Main Menu](https://github.com/seth586/guides/blob/master/README.md) ]

[ [Intro](README.md) ] - [ [Jail Creation](freenas_1_jail_creation.md) ] - [ [Bitcoin](freenas_2_bitcoin.md) ] - [ [Tor & i2p](freenas_3_tor.md) ] - [**Electrum**] - [ [lnd](freenas_5_lnd.md) ] - [ [loopd ](freenas_5a_loopd.md)] - [ [RTL](freenas_6_rtl.md) ] - [ [mempool](freenas_8_mempool.md) ] - [ [Extras](extras.md) ]

## TrueNASnode - full bitcoin stack deployment guide ![BSDBTC60.png](images/BSDBTC60.png)

Join the chatroom on the matrix chat protocol: [#truenasnode:nym.im](https://matrix.to/#/#truenasnode:nym.im)

### Electrs: Electrum In Rust

Read up more on electrs at its github page [here](https://github.com/romanz/electrs)

### 1. Set up environment
TrueNAS 12's base compiler is llvm10, however we need to downgrade to version 9 for a sucessful compile. Download prerequisites and compile:
```
# c++ --version
FreeBSD clang version 10.0.1
# pkg install rust git llvm90 nano
# ln /usr/local/llvm90/bin/clang++ /usr/local/llvm90/bin/c++
# nano ~/.cshrc
```
Add the `/usr/local/llvm90/bin` directory to a priority position in the shell's path:
```
...
set path = (/usr/local/llvm90/bin /sbin /bin /usr/sbin /usr/bin /usr/local/sbin /usr/local/bin $HOME/bin)
...
```
Save (CTRL+O, ENTER) and exit (CTRL+X). Refresh your shell:
```
# source ~/.cshrc
# c++ --version
clang version 9.0.1
```
Success! 

### 2. Compile & Install
```
# cd ~
# git clone https://github.com/romanz/electrs
# cd electrs
# cargo build --release
```

If you get an error, see this [issue for FreeBSD systems](https://github.com/romanz/electrs/issues/132#issuecomment-481870879) to fix. Rerun `cargo build --release`

Install and cleanup:
```
# install -m 0755 -o root -g wheel /root/electrs/target/release/electrs /usr/local/bin
# rm -r ~/electrs
# mkdir /var/db/electrs
```
### 3. Create RPC credentials for Bitcoin Core

Download the rpcauth tool as documented [here](https://github.com/bitcoin/bitcoin/tree/master/share/rpcauth). Save this information.

```
# pkg install python39
# fetch https://raw.githubusercontent.com/bitcoin/bitcoin/master/share/rpcauth/rpcauth.py
# python3.9 ./rpcauth.py electrs
String to be appended to bitcoin.conf:
rpcauth=electrs:5d0d70936350d0a79b588a9bb2906ea1$82afc2d29dfcfd808acd98f855cf47989564d8f1cd55b515f23fb10ace0dd75a
Your password:
2tm5NiN8wZVyjx_hgUL5O8it68WfoadHDEZ-v6w_RhQ=
```

Add the `rpcauth=` string above to `bitcoin.conf` and configure rpc access. Make sure that the `rcpallowip=` coorelates to your local subnet address range.
```
# nano /usr/local/etc/bitcoin.conf
rpcauth=electrs:5d0d70936350d0a79b588a9bb2906ea1$82afc2d29dfcfd808acd98f855cf47989564d8f1cd55b515f23fb10ace0dd75a
rpcallowip=192.168.84.0/24
rpcbind=0.0.0.0
```
Save (CTRL+O,ENTER) and exit (CTRL+X)

Reboot bitcoin core, make sure bitcoind is running sucessfuly after the reboot by running `ps aux`.
```
# service bitcoind restart
# ps aux
USER      PID %CPU %MEM     VSZ     RSS TT  STAT STARTED      TIME COMMAND
bitcoin 76206  4.4  2.2 4551716 1449288  -  SJ   Wed23   426:49.66 /usr/local/bin/bitcoind -conf=/usr/local/etc/bitcoin.conf -datadir=/var/db/bitcoin
...

```
### 4. Create user & config
See config notes [[here]](https://github.com/romanz/electrs/blob/master/doc/config_example.toml)
```
# pw adduser electrs -d /nonexistent -s /usr/sbin/nologin
# mkdir /var/db/electrs
# mkdir /usr/local/etc/electrs
# nano /usr/local/etc/electrs/config.toml
```

Use the `username:password` generated by `rpcauth.py` for the `auth =` field:

If you want to serve local home network connctions in addition to tor connections, replace `electrum-rpc-addr=` with your bitcoin jail's IP. You will need to change your tor `/usr/local/etc/tor/torrc` config `HiddenServicePort` IP address to your bitcoin jail's IP address as well   

```
jsonrpc_import = true
daemon_rpc_addr = "127.0.0.1:8332"
auth = "electrs:2tm5NiN8wZVyjx_hgUL5O8it68WfoadHDEZ-v6w_RhQ="
db_dir = "/var/db/electrs"
network = "bitcoin"
electrum_rpc_addr = "127.0.0.1:50001"
log_filters = "INFO"
```
Save (CTRL+O,ENTER) and exit (CTRL+X)

### 5. Create permissions & test run
```
# chown -R electrs:electrs /var/db/electrs
# chown -R electrs:electrs /usr/local/etc/electrs
# chmod -R 500 /usr/local/etc/electrs
# su -m electrs -c 'electrs --conf=/usr/local/etc/electrs/config.toml --skip-default-conf-files'
```

Electrs should begin to index the blockchain into its own database.  Stop the app with Ctrl+C.

### 6. rc.d script

```
# nano /usr/local/etc/rc.d/electrs
```

Paste the following script:
```
#!/bin/sh
#
# PROVIDE: electrs
# REQUIRE: bitcoind
# KEYWORD:

. /etc/rc.subr

name="electrs"
rcvar="electrs_enable"
electrs_command="/usr/local/bin/electrs --conf=/usr/local/etc/electrs/config.toml"
pidfile="/var/run/${name}.pid"
command="/usr/sbin/daemon"
command_args="-P ${pidfile} -u electrs -r -f ${electrs_command}"

load_rc_config $name
: ${electrs_enable:=no}

run_rc_command "$1"
```
Save, (ctrl+o,enter) and exit (ctrl+x)

Make the script executable:
```
# chmod +x /usr/local/etc/rc.d/electrs
```
And enable on startup:
```
# sysrc electrs_enable="YES"
```
Give it a whir:
```
# service electrs start
```

Electrs will begin to index the blockchain into its own database. This can take a few hours, depending on your CPU and disk IO. When its done indexing, it will start to serve connections.

### 7. Client Setup
Right click on your windows electrum client, select properties, and modify the shortcut (use your .onion address or bitcoin jail ip, if configured in step 4)
```
"C:\Program Files (x86)\Electrum\electrum-3.3.4.exe" -1 -s myprivateonionaddressocyn4rixm632jid.onion:50001:t
```
Start your tor browser to connect to the tor network. Start electrum, select /tools/network/proxy and enable `use tor proxy at port 9150`. You should connect!


### How to update electrs
```
# service electrs stop
# pkg update && pkg upgrade rust
# cd ~
# rm -r electrs
# git clone https://github.com/romanz/electrs
# cd electrs
# cargo build --release
# install -m 0755 -o root -g wheel /root/electrs/target/release/electrs /usr/local/bin
# rm -r ~/electrs
# service electrs start && tail -f /var/db/electrs/bitcoin/LOG
```

### Bonus: Reverse proxy configuration for domain & SSL certificate access
If you want to access electrum without using a VPN or TOR, you can have a SSL encrypted connection by configuring your [reverse proxy](https://github.com/seth586/guides/blob/master/FreeNAS/webserver/6_reverse_proxy.md) config file by appending the data below to the bottom of `/usr/local/etc/nginx/nginx.conf` in your reverseproxy jail:
```
### ELECTRUM.EXAMPLE.COM
stream {
        upstream electrs {
                server 192.168.84.21:50001;
        }

        server {
                listen 50002 ssl;
                proxy_pass electrs;
                ssl_certificate /usr/local/etc/letsencrypt/live/example.com/fullchain.pem;
                ssl_certificate_key /usr/local/etc/letsencrypt/live/example.com/privkey.pem;
                ssl_session_cache shared:SSL:1m;
                ssl_session_timeout 4h;
                ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
                ssl_prefer_server_ciphers on;
        }
}
### / ELECTRUM
```

Change `192.168.84.21` with the jail hosting electrum. Then add `load_module /usr/local/libexec/nginx/ngx_stream_module.so;` to the very first line of `nginx.conf`. Save (CTRL+O, ENTER) and exit (CTRL+X), and refresh nginx with `service nginx restart`

Next: [ [lnd](freenas_5_lnd.md) ]

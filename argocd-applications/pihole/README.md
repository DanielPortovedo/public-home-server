# How to setup PIHOLE on port 53 in ubuntu

Due to the fact that 'systemd-resolved' is running on port 53, we need to disable its use on this port: https://unix.stackexchange.com/questions/676942/free-up-port-53-on-ubuntu-so-custom-dns-server-can-use-it

1. Disable the stub listener

```
sudo nano /etc/systemd/resolved.conf

[Resolve]
DNSStubListener=no
```

2. Restart systemd-resolved

```
sudo systemctl restart systemd-resolved
```

3. Fix /etc/resolv.conf (critical)

```
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

## HOW TO RECOVER

1. Re-enable the stub listener

```
sudo nano /etc/systemd/resolved.conf

[Resolve]
DNSStubListener=yes
```

2. Restore resolv.conf

```
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

3. Restart systemd-resolved

```
sudo systemctl restart systemd-resolved
```
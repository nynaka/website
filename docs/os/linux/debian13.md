Debian Linux 13
===

## 設定

### mDNS

```bash
sudo apt install avahi-daemon
sudo systemctl start avahi-daemon
sudo systemctl enable avahi-daemon
```

### SSH

```bash
sudo apt install ssh
sudo systemctl start ssh
sudo systemctl enable ssh
```
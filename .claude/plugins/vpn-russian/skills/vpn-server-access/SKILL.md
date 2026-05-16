---
name: vpn-server-access
description: >-
  Connect to VPN VPS servers over SSH with paramiko (ed25519 key ~/.ssh/vpn_servers),
  run remote commands, write files via Python/base64, and run x-ui diagnostics.
  Use for any VPN server remote operations.
---

# VPN Server SSH Access

## SSH key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vpn_servers -C "my-vpn-servers"
ssh-copy-id -i ~/.ssh/vpn_servers.pub root@<IP>
```

`~/.ssh/config` entry:

```
Host vpn-ru
    HostName <IP>
    User root
    IdentityFile ~/.ssh/vpn_servers
    IdentitiesOnly yes
    ServerAliveInterval 120
    ServerAliveCountMax 2
```

## Paramiko pattern

```python
import paramiko, os

key = paramiko.Ed25519Key.from_private_key_file(os.path.expanduser("~/.ssh/vpn_servers"))
c = paramiko.SSHClient()
c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
c.connect("<IP>", username="root", pkey=key, timeout=20)

def run(cmd, timeout=15):
    stdin, stdout, stderr = c.exec_command(cmd, timeout=timeout)
    out = stdout.read().decode()
    err = stderr.read().decode()
    rc = stdout.channel.recv_exit_status()
    return rc, out, err

rc, out, err = run("systemctl is-active x-ui")
c.close()
```

## Write files (no SFTP)

```python
content = '{"key": "value"}'
run(f"""python3 -c "
with open('/path/to/file', 'w') as f:
    f.write({repr(content)})
print('written')
" """)
```

## Safe JSON writes (base64)

```python
import base64, json

data = json.dumps(my_dict, indent=2)
encoded = base64.b64encode(data.encode()).decode()
run(f"""python3 -c "
import base64, json
data = base64.b64decode('{encoded}').decode()
json.loads(data)
with open('/path/to/file.json', 'w') as f:
    f.write(data)
print('written ok')
" """)
```

## Harden new server (once)

```bash
ssh root@<IP> "
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config
systemctl restart sshd
"
```

## Diagnostics

```python
run("ss -tlnp | grep :443")
run("ufw status")
run("journalctl -u x-ui --no-pager -n 30")
run("ps aux | grep xray | grep -v grep")
```

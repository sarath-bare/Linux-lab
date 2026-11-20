# Team S:wqerver Runbook

## Phase 1: Users and groups

**Purpose:** create `devs` and `ops` groups and the team accounts.
**Run as:** a normal admin user with sudo (never as root).

### Procedure
```bash
sudo groupadd devs
sudo groupadd ops
sudo useradd -m -s /bin/bash -g devs alice
sudo useradd -m -s /bin/bash -g devs bob
sudo useradd -m -s /bin/bash -g ops nice
sudo useradd -m -s /bin/bash -g ops -G devs kane
# set a temporary password for each user with: sudo passwd <user>
for u in alice bob nice kane; do sudo chage -d 0 "$u"; done
```

### Verification
```bash
id alice; id kane          # kane: primary ops, supplementary devs
getent group devs ops
sudo chage -l kane | head -3   # "password must be changed"
```

### Rollback
```bash
for u in alice bob nice kane; do sudo userdel -r "$u"; done
sudo groupdel devs; sudo groupdel ops
```

### Notes
- Primary group is stored in /etc/passwd (4th field). Supplementary
  members are listed in /etc/group, so primary-only members don't show there.
- List all members of devs: `getent passwd | awk -F: '$4==1238'`
- `skb` already existed (never logged in) and is not part of this team.
- Temporary passwords are shared out-of-band, never written here.

Your lines about least privilege and accountability go into NOTES.md.

Next steps
Create and prove the service account:
bash
sudo useradd -r -s /usr/sbin/nologin svc-backup
getent passwd svc-backup      # shell should be /usr/sbin/nologin
sudo su - svc-backup          # should print: This account is currently not available.

purpose : is to create a user that cant login 
command : sudo useradd -r -s /usr/sbin/nologin svc-backup
verification: getent passwd svc-backup      # shell should be /usr/sbin/nologin
sudo su - svc-backup          # should print: This account is currently not available.



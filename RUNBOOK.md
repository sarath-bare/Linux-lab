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

## Phase 2: Shared project directory

**Purpose:** a folder where `devs` collaborate, files inherit the group,
and users cannot delete each other's files.
**Run as:** a normal admin user with sudo.

### Procedure
```bash
sudo mkdir -p /srv/projects/shared
sudo chown root:devs /srv/projects/shared
sudo chmod 3770 /srv/projects/shared
sudo apt install acl
sudo setfacl -d -m g:devs:rwx /srv/projects/shared
```

### Verification
```bash
ls -ld /srv/projects/shared          # drwxrws--T+
getfacl /srv/projects/shared         # default:group:devs:rwx
sudo -iu kane bash -c 'touch /srv/projects/shared/k.txt; ls -l /srv/projects/shared/k.txt'   # group devs
sudo -iu bob bash -c 'rm -f /srv/projects/shared/k.txt'   # Operation not permitted (sticky bit)
sudo -iu nice bash -c 'cd /srv/projects/shared'           # Permission denied (others have no access)
```

### Rollback
```bash
sudo rm -rf /srv/projects/shared
```

### Notes
- setgid (`s`): new files inherit the directory's group.
- sticky bit (`T`): only the owner of a file can delete it.
- Default umask 022 gave group read-only files, which is why the default ACL was added.

### Known issue
- bob could not append to a file created by kane (`>> k.txt` gave
  Permission denied) even though the group and ACL looked correct.
  Cause not yet found. Investigate later.

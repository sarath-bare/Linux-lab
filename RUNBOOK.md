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

## Phase 3: Limited sudo for the ops group

**Purpose:** let members of `ops` restart the cron service, and nothing else.
**Run as:** a normal admin user with sudo.

### Procedure
```bash
sudo visudo -f /etc/sudoers.d/ops
```
Add this single line, then save:
```
%ops ALL=(root) /usr/bin/systemctl restart cron
```
`visudo` checks the syntax before saving and sets the file to
owner root, mode 0440.

### Verification
```bash
sudo visudo -c                   # /etc/sudoers.d/ops: parsed OK
sudo -l -U kane                  # lists only: (root) /usr/bin/systemctl restart cron
sudo -l -U alice                 # not allowed to run sudo
sudo -iu kane
sudo systemctl restart cron      # works, exit status 0
sudo systemctl stop cron         # refused
sudo cat /etc/shadow             # refused
sudo -i                          # refused
```

### Rollback
```bash
sudo rm /etc/sudoers.d/ops
sudo visudo -c
```

### Notes
- Rules name the exact command with its full path.
- Never allow editors (vim), shells, less or wildcards in sudo rules:
  they can launch a shell and give full root access.
- Files in /etc/sudoers.d/ whose names contain a "." are silently ignored.
- Sudo denies by default: only commands named in a rule can run.
- Every attempt, allowed or refused, is logged (journalctl _COMM=sudo).

## Phase 4: Service accounts and password aging

### 4a. Service account

**Purpose:** an account for running a program (for example a backup job)
that nobody can log in to interactively.

#### Procedure
```bash
sudo useradd -r -s /usr/sbin/nologin svc-backup
```

#### Verification
```bash
getent passwd svc-backup          # UID below 1000, shell /usr/sbin/nologin
ls -ld /home/svc-backup           # no such directory (-r creates no home)
sudo -u svc-backup whoami         # works: prints svc-backup
sudo -iu svc-backup               # refused: This account is currently not available.
```

#### Rollback
```bash
sudo userdel svc-backup
```

#### Notes
- `-r` creates a system account (UID below 1000, no home directory).
- `nologin` blocks interactive logins (su, ssh, console) but not programs
  running as the account. `sudo -u svc-backup <command>` still works,
  so access to sudo itself must be controlled.
- UID ranges: 0 root, 1-999 system and service accounts, 1000 and up people.

### 4b. Password aging with chage

**Purpose:** enforce password expiry rules and control account lifetime.

#### Procedure
```bash
sudo chage -d 0 <user>            # force a password change at next login
sudo chage -M 90 -m 1 -W 14 <user>  # max 90 days, min 1 day, warn 14 days
sudo chage -E 2026-12-31 <user>   # account itself expires on this date
sudo chage -E -1 <user>           # remove the account expiry
```

#### Verification
```bash
sudo chage -l <user>
sudo passwd -S <user>             # L = locked, P = usable password
```

#### Rollback
```bash
sudo chage -M 99999 -m 0 -W 7 -E -1 <user>   # back to the defaults
```

#### Notes
- Password expiry (user must change it) is different from account expiry
  (no login at all). Contractor accounts usually get an account expiry date.
- `passwd -l` locks the password. `passwd -u` unlocks it.
- A lab test: password set 2026-11-23 with `-M 90` expired on 2027-02-21.

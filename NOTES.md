Accountability. Sudo logs who ran which command. If three admins share the root login, you can’t tell who did what.

:Removing -m doesn’t block login. Without -m the user just has no home directory. If they have a password and a valid shell, they can still log in, and they’d land in / with a warning. A missing home directory is an inconvenience, not a lock.

What actually blocks login:

bash
sudo useradd -r -s /usr/sbin/nologin svc-backup
Flag	Effect
-s /usr/sbin/nologin	the shell is a program that prints “This account is currently not available” and exits, so interactive login is refused
-r	creates a system account: UID below 1000, no home directory by default, no password aging

# Got a shell: now what?

## Stabilise (Linux)
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z, then on your machine:
stty raw -echo; fg
# back in the shell:
export TERM=xterm-256color
```

## Orient
```bash
id; hostname; uname -a; cat /etc/os-release
ip a; ss -tulpn
ps aux
```

## Quick privesc checks (Linux)
```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab; ls -la /etc/cron*
ls -la /home /opt /var/www /tmp
env; history
```

## Look for credentials
- Config files, `.env`, web app config, database connection strings
- `~/.bash_history`, `~/.ssh/`
- Backups and old files in web roots

## Windows basics
```powershell
whoami /all
systeminfo
net user; net localgroup administrators
ipconfig /all; netstat -ano
```

## My additions
(Add commands here as boxes teach you things. Note which box taught you each one.)

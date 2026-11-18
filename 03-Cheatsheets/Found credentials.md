# Found credentials: where else do they work?

Check each service on every host in scope:
- [ ] SSH
- [ ] SMB
- [ ] WinRM
- [ ] RDP
- [ ] MSSQL / MySQL / PostgreSQL
- [ ] Web admin panels, CMS logins
- [ ] FTP
- [ ] Other users on the same host (`su`, `sudo`)

Also try:
- Password reuse with variations (same user, other hosts)
- Username formats (`user`, `DOMAIN\user`, `user@domain`)

Before spraying, check the lockout policy. In a real engagement, lockouts hurt the client.

## My additions

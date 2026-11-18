
- Nmap initial scan `nmap -sV --open -oA getsimple_initial_scan 10.129.177.69`

![[Pasted image 20261104194100.png]]

- The initial scan is showing
	- Port `22/tcp` open, running `SSH` on `OpenSSH 8.2 Ubuntu 4ubuntu0.1`
	- Port `80/tcp` open, running HTTP on Apache httpd 2.4.41 (Ubuntu)
	- OS : Linux 
	
- Nmap full scan `nmap -p- --open -oA getsimple_full 10.129.177.69`

![[Pasted image 20261104195438.png]]

- Run `nc` to do some banner grabbing to confirm Nmap's findings. 

![[Pasted image 20261104195201.png]]

**Running Nmap script : `nmap -sC -p 22,80 -oA getsimple_script_scan 10.129.177.69`**

![[Pasted image 20261104195759.png]]

- This script is show the availability of the ssh-hostkey 
- The script has reviewed two new findings `http-robots.txt` file and the `/admin/` directory. 

**Running Nmap http-enum script : `nmap -sV --script=http-enum -oA getsimple_nmap_http_enum 10.129.177.69`**

![[Pasted image 20261104200513.png]]

- The script discovered more directories;
	- /admin/: Possible admin folder
	- /admin/index.php: Possible admin folder
	- /backups/: Backup folder w/ directory listing
	- /robots.txt: Robots file
	- /data/: Potentially interesting directory w/ listing on 'apache/2.4.41 (ubuntu)


## Web Footprinting 

- Running the `whatweb` :whatweb 10.129.177.69

![[Pasted image 20261104201352.png]]

- There was an error opening the https://10.129.177.69 
- Showing javascript and the GetSimple Title.

- Running a `curl http://10.129.177.69`

![[Pasted image 20261104201622.png]]

- This confirms that the website is powered by GetSimple CMS

![[Pasted image 20261104201941.png]]

- This shows more tech now: `HTML5, JavaScript, jQuery`.

- Run Gobuster to be thorough and check for any other accessible pages/directories; `gobuster dir -u http://10.129.177.69/admin/ --wordlist /usr/share/seclists/Discovery/Web-Content/common.txt`

![[Pasted image 20261104203811.png]]

- The gobuster shows some interesting files and directories 

![[Pasted image 20261104210534.png]]

- Loading http://10.129.177.69/data/ 


![[Pasted image 20261104210700.png]]


- Loading http://10.129.177.69/data/users/

![[Pasted image 20261104210836.png]]

- Loading the admin.xml file

![[Pasted image 20261104210932.png]]

- The file is showing the admin credentials:
	- USR: admin 
	- PWD:  d033e22ae348aeb5660fc2140aec35850c4da997 (encrypted)

![[Pasted image 20261104211744.png]]

- Using the hasid command, it's showing it's a `SHA-1` hash function

![[Pasted image 20261104212242.png]]

- On research `admin` is the encrypted password. 

- The cache file : http://10.129.177.69/data/cache/ is showing the below and it might be exploitable. 
`{"status":"0","latest":"3.3.16","your_version":"3.3.15","message":"You have an old version - please upgrade"}`

- The file http://10.129.177.69/data/other/authorization.xml is showing the `apikey` below: 

![[Pasted image 20261104213311.png]]

- The discovered credentials successfully logged into the dashboard.

![[Pasted image 20261104213845.png]]

- Admin dashboard 

![[Pasted image 20261104214033.png]]


- We have successfully logged into the dashboard. 

## Initial Foothold 



| Pages   | creating and editing pages       |
| ------- | -------------------------------- |
| Files   | uploading files                  |
| Theme   | Change the CMS theme             |
| Backups | Store backup pages               |
| Plugis  | Installed and deactivate plugins |

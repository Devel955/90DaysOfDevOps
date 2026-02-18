 # day 3 Linux Commands Cheat Sheet
 # Process Managment:
 ps-aux : sab running process dikhata hain.
 top- live CPU/RAM check karta hain.
 Kill PDI - Process band karta hain.
 systemctl status nginx - services ka status.
 systemctl restart nginx — restart a service.
 journalctl -u nginx -n 50 — last 50 logs for a service.


# File system:
ls- list file
pwd - current location
cd folder - folder change
mkdir - folder banana
rm file - file delete
tail -f app.log -live dikhata hain.
grep ERROR log.txt - error 
'pwd' - print current directory path.
chown user: group file - change file owner/group.
`mkdir -p a/b/c` — create nested directories.

# Networking troubleshooting:
ip addr - IP check
ping google.com - internet check
curl -I site.com - website check
ss-tuplen - port checks 

# HTB - NFS Enumeration (Practical Commands)

Category: Enumeration
Command: nmap -sV -p111,2049 <IP>
nmap --script nfs* -p111,2049 <IP>
showmount -e <IP>
mkdir target-nfs
sudo mount -t nfs <IP>:/ ./target-nfs -o nolock
cd target-nfs
ls
ls -R
ls -l
ls -n
cat flag.txt
cd ..
sudo umount ./target-nfs
Description: NFS (HTB) – quick workflow
1) Scan: nmap -sV -p111,2049 <IP>
2) Enum exports: nmap --script nfs -p111,2049 <IP>
3) List shares: showmount -e <IP>
4) Mount: sudo mount -t nfs <IP>:/ ./target-nfs -o nolock
5) Explore: ls / ls -R / ls -l / ls -n
6) Read: cat flag.txt
7) Unmount: sudo umount ./target-nfs

10 commands (SMB + NFS)
SMB
1. nmap -sV -sC -p139,445 <IP>
2. smbclient -L //<IP> -N
3. rpcclient -U "" <IP> -N
4. rpcclient -U "" <IP> -N -c "srvinfo"
5. rpcclient -U "" <IP> -N -c "querydominfo"
6. rpcclient -U "" <IP> -N -c "netshareenumall"
7. rpcclient -U "" <IP> -N -c "netsharegetinfo <share>"

NFS
8. nmap --script nfs -p111,2049 <IP>
9. showmount -e <IP>
10. sudo mount -t nfs <IP>:/ <mountdir> -o nolock
Tags: enumeration, nmap
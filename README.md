# Dumping ZTE ZXHN H168N V2.2 Files


## Acessing Router Admin Portal

Admin panel can be accessed at 192.168.10.1 by default
```
Username    : admin 
Password    : <Written at bottom of router>
```

## Router Configuration

Router configuration can be downloaded by going to Management>Settings>Backup

By cliking on Backup Setting config.bin file can be downloaded. But the file is encrypted.


## Decrypting Binary File

The config.bin file can can decrypted using https://github.com/mkst/zte-config-utility tool

Required libraries can be installed using 
```sh
pip install -r requirments.txt && python3 setup.py install --user
```

To decrypt binary run 
```sh
python examples/decode.py ./config.bin ./config.xml
```

Executing the command would show signature, payload type and the key that was used to decode the binary file

```Detected signature: ZXHN H168N V2.2
Detected payload type 2
Successfully decoded using key: 'Q#Zxn*x3kVLc'!
```

## Examining the contents of XML File

First thing that need to be examined are credentials for ftp, ssh, and telnet as those ports are open (can be found using nmap scan)

At line 1453 different credentials for different services are listed e.g. AppID 1 is for web service,
AppID 2 is for telnet, AppID 4 is for ftp and so on.

Using telnet credentials root and public opens up a cli. Typing ? in terminal lists available commands. Using enable command and by typing password "zte" new terminal opens with more commands through which router can be configured. 
(If this router's firmware wasn't modified ZTE router by ISP there could have been another command like shell which would give root shell into router).

At line # 3657 in xml config file there are credentials for ssh but using those credentials also bears same result as telnet as it is executing one of binaries inside router which will be looked upon later in the write-up.

Last thing that remains is ftp. Using the credentials from configuration file ftp folder can be accessed, but it is empty and directories can't be changed. 

At line # 2530 in config file. FTP tries to mount Location /mnt which is an empty directory. 

<Tbl name="FTPUser" RowCount="1">
<Row No="0">
<DM name="ViewName" val="IGD.FTPUSER0"/>
<DM name="Username" val="admin"/>
<DM name="Password" val="admin"/>
<DM name="Location" val="/mnt"/>
<DM name="UserRight" val="3"/>
</Row>

What if Location is changed from /mnt to / ? Will it mount whole operating system? 
By changing Location from /mnt to / . It can be repackaged now.

## Encoding config.xml

Using same zte config utility the new config file can be repackaged and uploaded to router.
```
python examples/encode.py --signature 'ZXHN H168N V2.2' --key 'Q#Zxn*x3kVLc' --include-header newconfig.xml newconfig.bin
```

By going to Management>Settings>Backup new config binary can be uploaded. After router reboots new settings will take effects.

Now after logging in using FTP whole system files can be accessed but still as user with low privileges some files can't be accessed and uploaded, and it is also not proper shell. 

## Dumping Files
To make life easier dumping files instead of using ftp, lftp was used to download all files recursively. After logging in using lftp, mirror command can be used to dump each folder. 

e.g. 
```mirror /bin /name_of_local_folder/bin```
Some directories like /dev would stop working and command would be stuck, so it is advised to dump each directory separately.

## cliagent binary 

When trying to use ssh instead of getting a shell, it instead greeted us with this text.
```

          ************************************************************
                          Welcome to the world of CLI !
          ************************************************************
```
It was both same for Telnet and SSH. It means both are trying to execute same binary when shell was invoked. The binary can be found in /bin directory with name of cliagent. 

Inspecting this binary in Ghidra reveals same username and password that were used to get simple CLI shell.

## NOTES

- There are lots of interesting files specially webserver lua files in /home directory
- Due to my limited skill with reverse engineering I didn't look more into cliagent and how it was modified to disable shell access.

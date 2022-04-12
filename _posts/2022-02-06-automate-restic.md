---
layout: post
title: Fully automated, dirt-cheap, encrypted backups
categories: [syncthing, restic, backup, s3, b2, syncthing]
---

Do you worry you might lose your personal files once and for all? Want a set-and-forget solution that's also dirt cheap? In ~60 minutes you will never have to worry no more. Meet `restic` + `syncthing` + Backblaze B2. 😌

Almost everyone has some files they cannot afford to lose. Photos, diaries, business files, the list goes on. Yet so few people take proper care to protect them from disaster scenarios. Why? Because backups are not fun. But we need them anyway. So let's stop complaining and start configuring!

# What you'll get

- **Automated backups**. Set this up once and never touch it again. Unless you need to restore your precious backups, of course.
- **Off-site backups**. A copy of your backup will reside in the cloud. Your house might burn down but your files will not.
- **Encrypted backups**. Only you can read those files. Lose your physical devices with confidence (please who steals one of your backup devices can access your files. Noone who gets access to your cloud storage can access your files.
- **Cheap backups**. This will probably cost you less than 1 € a month. You might need to buy a memory card for your phone (~ 30 € for 256 GB).
- **Go back way in time**. You will have a backup of yesterday, last week, last month and even last year. Pretty hard to lose something for good.
- **Malware protection**. Something deleted your files? Something encrypted your files and wants a ransom? Just restore that backup from yesterday, last week, last month, or heck even last year if you need to. No need to pay that ransomware.

Plus some nice goodies:

- **No need to trust anyone**. All software we use is free, open-source code.
- **No vendor lock-in**. Well that's pretty much a consequence of that first point.
- **Access your files anywhere**. You will have an offline copy of everything, so no need for internet or making files available offline before leaving the house.

You've heard it before, and I will tell you again: You need a backup of the files you cannot afford to lose.

Let's face it: Nobody likes backups. And why should we? It's not that our devices but everybody needs them.

For many years I have used a chaotic mix of Google Drive, ownCloud and offline storage to kind of "back up" my personal files. I had the Windows desktop clients for those cloud storages that synced between my computer and the cloud. I liked the convenience of accessing my files anywhere I went--well at least those that were small enough to store in Drive and as long as I had internet--and I could not really imagine that Google would ever lose my files.

On my quest to backup my 100 GB of personal files once and for all so that nothing could ever happen to them, I configured a [Syncthing](https://syncthing.net/) network of 3 devices: My laptop (has a big enough disk), my phone (actually chose one with 256 GB just for this) and my old phone ([configured an extra SD card as internal storage](https://www.nextpit.com/how-to-format-microsd-cards-as-internal-storage)). My phone is always online, so that whenever I power on one of the other two devices everything will be synced. So nothing more to worry about, right?

Well, kinda. It might seem unlikely, but my backup network was still vulnerable to something that would overwrite a file again

Having configured my Syncthing network of my laptop, my phone and my second, old phone

This is a guide on how to set up a Linux virtual private server (VPS) for running a [Syncthing](https://syncthing.net/) node. After completing this guide, you will have a VPS serving as a Syncthing node in your Syncthing network, and which is hardened against attacks.

We will discuss

1. Renting a VPS
2. Setting up and restricting SSH access
3. Running Syncthing with Docker
4. Encrypting your synced data

# What is a VPS?

VPS (or V-Server) is short for virtual private server. It's a virtual machine that shares physical resources with other virtual machines.

Common operating systems for VPSs are Linux or Windows. In this guide, we will be using a VPS running RedHat's CentOS 8. If you prefer CentOS 7, use `yum` whenever you should use `dnf`.

We will refer to our VPS as _remote_.

# Where do I get one?

You can rent a VPS with your favorite provider. I recommend [Hetzner](https://www.hetzner.com) because of their admin panel. You can also try out several providers and see which one you like the most. They usually offer a full refund when cancelling within a few weeks.

Some providers offer easy ways to upgrade your plan, so you can start with the cheapest one and upgrade when you need more computing power or space.

Note that it might take a day or two for your provider to set up your VPS.

# [i@localhost ~]$ and [root@remote ~]\#

If you are new to Linux you might be wondering what these these things in front of the actual commands mean. They appear when your console prompts you for a command.

`[i@localhost ~]$` indicates that the command will be executed by user `i`, on machine `localhost`, in the `~` directory and not as `root` (`$`).

`[root@remote ~]#` indicates that the command will be executed by `root`, on machine `remote`, in the `~` directory and as `root` (`#`).

To save some keystrokes when you must execute one or more commands `command1`, ..., `commandN` as `root`, instead of prefixing each with `sudo` like so

```console
[i@localhost ~]$ sudo command1
  ⋮
[i@localhost ~]$ sudo commandN
```

you can enter the interactive mode first

```console
[i@localhost ~]$ sudo -i
[root@localhost ~]# command1
  ⋮
[root@localhost ~]# commandN
```

# Set up access via SSH

Throughout this guide, we will be logged in as user `i` on our local machine `localhost` and connect to our remote machine `remote` via SSH. Your provider might already have set your remote to listen for incoming SSH connection requests. In case they did not, connect to your remote through their web interface and start and enable the service daemon `sshd`:

```console
[root@remote ~]# systemctl enable --now sshd
```

You can now open an SSH session from your local machine to your remote. Look up the domain or IP your provider has assigned to your VPS. We will call it `<domain>`. Then:

```console
[i@localhost ~]$ ssh root@<domain>
```

You will be shown the ECDSA, RSA or ED25519 fingerprint `<fingerprint>` of your remote which should look like `SHA256:r4nd0Mch4Rac73r5`. Make sure it is the same as when calculating it on your remote:

```console
[root@remote ~]$ ssh-keyscan localhost | ssh-keygen -lf - | grep <fingerprint>
```

If you don't see `<fingerprint>` highlighted in above command's output, your connection is compromised. Abort the connection!

If the fingerprint is correct, accept it. You will be prompted for `root`'s password. When successful, your shell will indicate that you are now logged in on your remote machine.

# Only allow authentication with public keys

Your remote currently allows SSH logins with passwords. The problem with passwords is that they can be brute-forced, and yes, people are systematically trying to do so. Luckily, we can require public keys for authentication. Public keys are more secure than passwords as brute forcing them is virtually impossible. To lock the bad guys out, let's set up public key authentication.

## Generate a key pair

On your local machine, generate a new SSH key for logging in as `root` on your remote:

```console
[i@localhost ~]$ ssh-keygen -f ~/.ssh/root@<domain> -C i@localhost
```

Your new key will be saved to `~/.ssh`. `ssh` will look for keys in that directory by default.

Once you get comfortable with using SSH keys, you will probably find yourself with a few more, so stay organized! Use `-f` to give your private and public key file sensible names (e.g. `joe@example.com` and `joe@example.com.pub`). Use `-C` to print a comment which others can make sense of to your public key file.

## Authorize your public key

We now need to let the remote machine know about our public key. The remote machine keeps authorized keys in `~/.ssh/authorized_keys`. An easy way to authorize our public key is to run

```console
[i@localhost ~]$ ssh-copy-id -i ~/.ssh/root@<domain> root@<domain>
```

This will send the key file `~/.ssh/root@<domain>` to the remote, add it to `root@remote`'s `authorized_keys` and set the appropriate file permissions.

If your prefer to do things manually, you can also pipe your public key through `ssh` into `authorized keys` with

```console
[i@localhost ~]$ cat ~/.ssh/root@<domain>.pub | ssh root@<domain> "cat >> ~/.ssh/authorized_keys"
```

or manually with copy and paste.

If you copied your public key these ways and logging in with your public key does not work, make sure that only `root` can write to `authorized_keys`. Use

```console
[root@remote ~]# ls -l ~/.ssh/authorized_keys
```

to check file permissions. Use

```console
[root@remote ~]# chmod go-wx ~/.ssh/authorized
```

to disallow the owning group and others to write the file.

## Configuring SSH on your remote

To handle incoming SSH connections, your remote will run a service daemon called `sshd`. It can be configured by editing `/etc/ssh/sshd_config`. There are a lot of configuration possibilites which you can check with `man sshd_config`. E.g.:

| key                                                          | description                                                                                                                                                                                                                                                                                                                                         | defaults to                                  |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `PubkeyAuthentication`                                       | Make sure this is `yes` to allow public keys as authentication method.                                                                                                                                                                                                                                                                              | `yes`                                        |
| `PasswordAuthentication`                                     | To prevent getting brute-forced, forbid password authentication by setting this to `no`.                                                                                                                                                                                                                                                            | `yes`                                        |
| `AuthenticationMethods`                                      | Use this to define what single authentication method or set of authentication methods must be successful. Can be used as an alternative to the two options above.                                                                                                                                                                                   | `any`                                        |
| `Port`                                                       | Specify which port to listen on for incoming SSH connections. Set to something other than the default `22` to make it harder (not impossible!) for the average malicious scanner to find your SSH server. Anyone who knows which port you are listening on can still attack you, so do not rely on security through obfuscation to secure your VPS! | `22`                                         |
| `Banner`                                                     | Specify a text file sent to the client prior to authentication. Use this to greet the user.                                                                                                                                                                                                                                                         | `none`                                       |
| `Ciphers`                                                    | Specify which ciphers you allow.                                                                                                                                                                                                                                                                                                                    | see `man sshd_config`                        |
| `ClientAliveCountMax`, `ClientAliveInterval`, `TCPKeepAlive` | Automatically close the session when the client stops responding.                                                                                                                                                                                                                                                                                   | `3`, `0`, `yes`                              |
| `TrustedUserCAKeys`, `RevokedKeys`                           | Authorize public keys signed by a specified certification authority (CA). Use this to administer public keys through your CA instead of `authorized_keys`. Revoke keys that have not expired yet.                                                                                                                                                   | see `man sshd_config`, see `man sshd_config` |

To only allow public key authentication, open the configuration on your remote with

```console
[root@remote ~]# vi /etc/ssh/sshd_config
```

and delete any `PubkeyAuthentication no` and `PasswordAuthentication yes` if present. Add a new line `PasswordAuthentication no`. Save the file.

Restart `sshd` to make it use the new configuration:

```console
[root@remote ~]# systemctl restart sshd
```

## But I do need password login!

If you must or want to use password authentication, make sure to block malicious requests. E.g., [fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page) bans clients who repeatedly fail to authenticate. You don't need fail2ban if you disable password login.

Install with

```console
[root@remote ~]# dnf install epel-release
[root@remote ~]# dnf install fail2ban
```

Detailed instructions on how to setup fail2ban can be found at `man jail.conf` at `/etc/fail2ban/jail.conf`. A basic setup that bans clients for 1 day after 5 failed attempts within the last hour, create `/etc/fail2ban/jail.local` and add the following content:

```conf
[DEFAULT]
bantime = 1d
findtime = 1h
maxretry = 5

[sshd]
enabled = true
```

If you configured `sshd` to listen on a non-default port `<yourport>` add the line `port = <yourport>` in your `sshd` jail.

Launch the service daemon now and after every boot with

```console
[root@remote ~]# systemctl enable --now fail2ban
```

Whenever you change your fail2ban configuration, you must relaunch the daemon with

```console
[root@remote ~]# systemctl restart fail2ban
```

## Block requests from certain countries

You might be tempted to block any requests that do not originate from your home country. However, location is easily spoofable, so it might not protect against attacks as much as you hope. There is also no plain solution that I would like to recommend, and [others note it is not a common setup](https://serverfault.com/a/598078). This might also cause trouble if you ever need to acces your remote while on vacation or while using a VPN.

# A user just for Syncthing

You will probably want your remote to be more than just a Syncthing node in your Syncthing network. Maybe you want to run a web server. Maybe you want to run a Syncthing node for a friend's Synthing network. In these cases you don't want no service to snoop into files that are none of their business.

We will create a user who will own the files we sync with Syncthing. If we follow this "one non-root user for every service that works with files" principle, we can be sure that none of them can snoop around if they ever get compromised.

Let's create a new user `syncthing` with sudo privileges:

```console
[root@remote ~]# adduser syncthing
[root@remote ~]# passwd syncthing
[root@remote ~]# usermod -aG wheel syncthing
```

You can run

```console
[root@remote ~]# id syncthing
```

to verify that `syncthing` belongs to the wheel group. Under CentOS, users in the wheel group may invoke `sudo`.

If you would like to sign in as `syncthing` via SSH, you will need to setup SSH access for this user as well. If you authorize your public key manually, make sure to add it to the new user's file, i.e. `/home/syncthing/.ssh/authorized_keys`.

# Install Docker

Since there is no CentOS repository for Syncthing, we will install Syncthing through Docker.

To install Docker, follow [the official documentation](https://docs.docker.com/engine/install/centos/).

To run Docker now and after every boot:

```console
[root@remote ~]# systemctl enable --now docker
```

# Run Syncthing

Create a dedicated network for Syncthing:

```console
[root@remote ~]# docker network create syncthing
```

Then pull its Docker image from [Docker Hub](https://hub.docker.com/r/syncthing/syncthing) and run it now and whenever the Docker daemon starts:

```console
[root@remote ~]# docker run --name syncthing --network syncthing --detach --env PUID=$(id --user syncthing) --env PGID=$(id --group syncthing) --volume /home/syncthing:/var/syncthing --publish 22000:22000 --publish 8384:8384 --restart always syncthing/syncthing
```

We call our container `syncthing` to easily reference it with commands like `docker stop syncthing`. We put it in the dedicated network to protect it from other running containers. `--detach` makes the container not use our console as standard output. We pass the user id of user `syncthing` and group id of group `syncthing` as environment variables so that Syncthing uses the appropriate user and group when syncing our files. `--volume` makes the synced files persist on our host. `--publish` forwards requests to the specified port on our host machine to the specified port on the container. `restart always` (also) makes the container start when Docker starts.

To access your remote's Synthing web interface from your local machine, use `ssh`'s port forwarding. E.g., if Syncthing is running on http://localhost:8384 on your remote, use

```console
[i@localhost ~]$ ssh -L 8385:localhost:8384 <domain>
```

to forward requests to http://localhost:8385 in your local browser to your remote machine. This way, your local Syncthing instance can still use the default location http://localhost:8384.

# Are my files secure on my remote?

If you only allow SSH login with public keys, your files are safe when it comes to people who try to break into your VPS from the outside.

However, strictly speaking **your files are not safe from your VPS provider** who has physical access! Because of that, they are also not safe from law enforcement or anyone gaining control over your provider. Even if you encrypt your files on your remote, they are technically still visible in plain while in memory.

If you do not want anyone ever to read your files without your consent, encrypt your files before syncing them with Syncthing. I personally encrypt my few most personal files with [gocryptfs](https://nuetzlich.net/gocryptfs/).

# Credits

I could not have written this guide without those many man pages and online documentations. Thank you very much to all those who contributed to them.

Also thank you very much to the authors of the following articles. Please take a look if you find some instructions here were not clear enough, as these are very beginner-friendly and go into much more detail.

- J. Ellingwood, [_How To Configure SSH Key-Based Authentication on a Linux Server_](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
- B. Bearnes, [_How To Protect SSH With Fail2Ban on CentOS 7_](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-centos-7)
- R. Shaw, [_Protect your system with fail2ban and firewalld blacklists_](https://fedoramagazine.org/protect-your-system-with-fail2ban-and-firewalld-blacklists/)
- V. Gite, [_Add / Create a Sudo User on CentOS Linux 8 sudoers_](https://www.cyberciti.biz/faq/add-create-a-sudo-user-on-centos-linux-8/)
- Syncthing contributors, [_Syncthing's Docker README_](https://github.com/syncthing/syncthing/blob/main/README-Docker.md)

Happy syncing! :tada:

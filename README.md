# pg-backup

Frustrated by the sheer complexity for backing up a PostgreSQL system and a lack of turnkey solutions, and a web full of patchy and poorly written tutorials  I took a crack at an as-turnkey-as-possible solution for my needs. I hope you find some use out of it without having to become a pro at databases.

## When is this Useful

To use these two scripts sensibly the following scenario is presumed. If it fits, good, if not, then keep looking:

- You have a PostgreSQL server (v12 or higher)
  - You have sudo access on that server
  - You want to back it up to a remote server
- You have password free (keyed) ssh access to that remote backup server 
  - or can arrange it

If these apply, then these two scripts are about as turn-key as you can get.

Of course the web is not full of tutorials and warnings it's complicated for no reason. There are few simple concepts you need to be on top of and feel comfortable with to call this remotely turn-key.

## Key Concepts

- **bash**:  These are bash scripts. Best to know what bash is and what a bash script is. Or you're in a little deep. 
- **Base backups**: A backup of the entire database. `pg_basebackup` handles this. You want to take a complete base backup at intervals like once a week, month or year depending on traffic and size and whatever else. Backing up the whole database can be a little slow and so day to day you want incremental backups.
- **Incremental backups**: Backing up the changes since the last base backup. In PostgreSQL parlance this is done by backing up Write-Ahead-Logs (WALs). You'll need to enable those, they aren't on by default. They are not on by default because there is of course always a performance hit to expect with every bit of extra work a system does, and WALs are PostgresSQL's log of changes. They are called Write-Ahead only because the change is logged before it's applied (as opposed to the other way around).
- **Keyed SSL access** (aka passwordless ssh): Up front, you want to know what ssh is. If that's new to you, you're on shaky ground here, but put simply it's a way of accessing a remote machine over a secure sockets layer (so, private and secure, and the norm today). Normally if were `ssh user@server.lan` you'd get a login prompt, login and then find yourself in a bash session. The key concept you'll need in prep here is setting this up so you use keys not passwords. 
  - That means generating a private key. `ssh-keygen -t rsa` which generates a private/public pair of keys for you and drops them in `~/.ssh`
  - Copying the public key to the server. 
    - Easiest is `ssh-copy-id -i ~/.ssh/id_rsa.pub user@server.lan`
    - If that's not available, this is fine too: `cat ~/.ssh/id_rsa.pub | ssh user@server.lan "cat >> ~/.ssh/authorized_keys"`
    - Really the public key just has to land on the backup server in the file `~/.ssh/authorized_keys` by whatever means.
    - Normally I'd permit only keyed access for such a service, set up a user `backup` no the target machine, with a shell and password, copy my private key over, then remove the password for that user on the target machine.
  - If the keys are in place (private and public on your database server) and public on the backup server (in the right places) then `ssh user@server.lan` should log you in with no password request. All keyed, and as secure as your private key (so keep it secure, it's in `~/.ssh/id_rsa` and should have like `rw-------` permission, legible only by you - well your user on the PostgreSQL server)
  - There's a catch here to be aware of here. Any time you do `sudo ssh user@server.lan` or `sudo -u postgres ssh user@server.lan` for example it will not work unless `~root` or `~postgres` respectively have your private key in their .ssh directory. `ssh` alas will look in the home directory of whoever you're masquerading as with sudo. This is a gotcha alas with sudo and keyed ssh.
- **sshfs**: We couldn't escape using this easily, but it's a system administrators dream come true. It means you can mount any directory you can access via keyed ssl (above) as a local directory. The `pg-backup` script uses three  PostgreSQL utilities (`pg_basebackup`, `pg_controldata` and `pg_archivecleanup`) that aren't remote server savvy. Specifically:
  - `pg_basebackup` performs the Base backup and can only drop the backup on a local file system
  - `pg_controldata` we use to get the PostgreSQL version and work out which WALs to keep or discard.
  - `pg_archivecleanup` essentially cleans up WALs that predate the base backup so they don't collect, like lint,
- **rsync**:  Really all you need to know is that it's  standard tool for archiving and copying data between servers, but alas it doesn't work well with sshfs mounts, so access the backup server itself opening another SSL connection, it is remote server savvy (unlike the PostggreSQL utilities)

# Using pg-backup and pg-restore

With key concepts down pat, you'll find this is just two, not overly large, bash scripts. The idea is simple enough:

1. Ensure you have ssh access to a remote backup server and can define the variables at the top of these scripts that declare the user, hostname and directory on that backup machine to use.  You'll need that. Difficult to back up to remote host you can't connect to. Alas the bar is a little high, you need ssh access which not all cloud  based backup providers will give you. So I use this where I can, and only LAN to back up PostgreSQL not another server and from there seek to replicate those backups on the cloud.

2. Assuming you have an account on the PostgreSQL server and can log in to it (ssh presumably)

   - Drop the two scripts into `~/bin`
   - Configure them (at top they have variables to configure)
     - Notably `rebase_days` defines the period between Base backups. Incremental backups are taken on every run. 

3. Test tpg-backup

   - `pg-backup -dv` is a good start for a verbose dry run. This is worth checking. It only does three things really:

     - Check for an existing Base backup and how old it is, and decide whether to do a Base backup, and if so use `pg_basebackup` to back it up.
     - Backup the WALs using `rsync`
     - Clean up the WALs, using `pg_archivecleanup` (it needs to know the oldest WAL file to keep which is revealed by `pg_controldata`)

     
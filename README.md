# iconcheck

iconcheck is a bash script for monitoring ICON-NMR sessions on mutiple NMR instruments
using [rsync](https://download.samba.org/pub/rsync/rsync.html). It copies IconDriverDebug 
and Inmracct files over from remote machines via SSH and checks them for new errors. If 
errors are found, it assembles them into log files and sends alert emails to the user and/or 
facility manager according to settings in the file input/error_table. The error_table file 
is meant to be customized by the user as new instrument-specific bugs inevitably emerge. 

This program requires that the local and remote computers have password-less SSH between them
enabled via private rsa keys.

I suggest running this as a cron job every five minutes or so.

## Prerequisites

This script requires a linux operating system with rsync. I've tested it in CentOS 6.8, 7.5 and x on the 
local side and CentOS 7.5, CentOS 5.1 on the remote side. I have only tested this with 
ICON-NMR run out of Topspin 2.1 and 4.0.8. 

The email feature requires that the command *sendmail* is working on the machine running the script.

## Installing
```sh
git clone https://github.com/greenwoodad/iconcheck
```
or 

```sh
git clone https://(your github username)@github.com/greenwoodad/iconcheck.git
```

followed by:
```sh
chmod +x ./iconcheck/iconcheck
```
## Getting Started

### Setting up password-less ssh logins to instrument machines

Because this script is intended to be run as a cron job, it is necessary to authorize the local
machine to access the remote machine(s) with password-less ssh login using * ssh keys *. Tutorials
are available here: 
* [How To Set Up Passwordless SSH Login](https://linuxize.com/post/how-to-setup-passwordless-ssh-login/)
* [OpenSSH Config File Examples](https://www.cyberciti.biz/faq/create-ssh-config-file-on-linux-unix/)

Briefly: 
1) On the machine you want to run the script and send emails from (as the user you want to do this as) run the command:

```sh
ssh-keygen -t rsa -b 4096
```

This will generate files ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub 

Press enter at the prompt "Enter passphrase (empty for no passphrase):" to skip passphrase generation.

2) Next, run this command (from the local machine) for each remote workstation:

```sh
ssh-copy-id remote_username@remote_ip_address
```
You will be prompted for the password for this remote workstation. 

If ssh-copy-id is not available, you should be able to run this instead:

```sh
cat ~/.ssh/id_rsa.pub | ssh remote_username@remote_ip_address "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

3) Last, add SSH aliases to your hosts file. In /etc/hosts, add entries:

```sh
IPAddress DomainName SSHAlias
```

for each remote workstation.

for example:
```sh
198.51.100.50     dmx500.chem.university.edu       DMX500
198.51.100.54     av400.chem.university.edu        AV400
198.51.100.59     neo400.chem.university.edu       NEO400
```
The SSHAliases here should be the same SSHAliases you enter in the input file. 

You should now be able to SSH to the remote workstations without entering a password by typing: 

```sh
ssh remote_username@SSHAlias
```

in addition to 
```sh
ssh remote_username@IPAddress 
```
and
```sh
ssh remote_username@DomainName 
```

The first time you do this, you will need to type "yes" to the question "Are you sure you want
to continue connecting (yes/no)?" however. After this, you will be able to run the script 
automatically without manual password entry.

### Configuring the input files

#### Main input file

In the main input file (iconcheck_input) there are a number of parameters and paths to set:

* `ScriptsPath`: Location of the main script and the input, emailtxt, and log folders on local machine.
* `SendmailPath`: Location of the sendmail application on local machine, probably /usr/sbin.
* `ManagerEmail`: Email address of the NMR facility manager.
* `Instrument`: Alias for password-less SSH to this instrument computer.
* `RemoteUser`: User on the remote computer that you can SSH as.
* `DebugPath`: Folder containing IconDriverDebug files on remote computer (probably something like /opt/topspin_version/prog/curdir/nmr1)
* `INMRPath`: Folder containing Inmracct.brief file on remote computer (probably something like /opt/topspin_version/conf/instr/spect/inmrusers)

IMPORTANT: When editing this file, do not use entries that contain spaces. 

#### error_table

The error table lists various ICON-NMR errors and determines how iconcheck will respond to them. The first column contains bits of the 
the error message as it appears in the IconDriverDebug file. (Note that in some cases special characters need to be escaped, such as `\"` 
to escape a double quote. Wildcards can be used with `.*` as well.) The second column points to the relevent email text that will then 
be sent, and the final two columns specify whether an email should be sent to the user, the NMR Manager, both, or neither. New entries 
can be added as new errors develop.

If you want to add a new entry, you should try to identify a unique string that reliably occurs in IconDriverDebug for this error (and 
not in other contexts) exactly once. Note that when adding a new error to the table, the script will submit emails corresponding to recent
instances of the error the next time it runs. To avoid this, you may choose to set `mail user?` to `n` temporarily until the script
adds the recent instances to the debugerrors log. 

IMPORTANT: When editing this file, make sure there are always at least two spaces separating entries for proper parsing of the input file.

#### addressbook

The address book provides email addresses for each ICON-NMR user. This is independent from the way ICON-NMR keeps track of email addresses
(in user files in conf/instr/instrument_name/inmrusers) but is not hard to set up. The file /input/addressbook shows some example entries. 
Each line should contain a username followed by that user's email address, separated by a space. If a user does not wish to receive emails,
it is possible to simply omit that user's entry in this file.

## Usage

The script can be run as follows:

```sh
iconcheck [OPTIONS]... /path/to/iconcheck_input
```

Options:

-h, -?, --help                           Show help message.

-i, --input                              Set input file (flag optional).

-e, --email (default y)                  Set to 'n' to skip sending emails

-t, --trim (default y)                   Set to 'n' to skip trimming IconDriverDebug.$Instrument.full


To run this as a cron job, make an entry in your crontab like this:

```sh
*/5 * * * * /path/to/iconcheck "/path/to/input/iconcheck_input"
```

## Contributing
Pull requests are welcome. 

## Authors

  - **Alex Greenwood** - *provided script* -
    [Greenwoodad](https://github.com/Greenwoodad)

## License
[MIT](https://choosealicense.com/licenses/mit/)

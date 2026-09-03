# iconcheck

iconcheck is a bash script for monitoring [ICON-NMR](https://www.bruker.com/en/products-and-solutions/mr/nmr-software/topspin.html) automation sessions on multiple NMR instruments. It serves as an alternative to the built-in ICON-NMR email feature, which is rather limited. 

The script uses [rsync](https://download.samba.org/pub/rsync/rsync.html) and SSH to periodically copy the `IconDriverDebug` and `Inmracct` log files from remote instrument computers, checks them for new errors, and sends alert emails to the user and/or facility manager according to settings in a customizable `error_table` file. New instrument-specific errors can be added to the error table as they inevitably emerge.

This program requires that the local and remote computers have password-less SSH between them
enabled via private rsa keys.

I suggest running this as a cron job every five minutes or so.

## Prerequisites

- Linux operating system (local machine)
- `rsync` and a sendmail-compatible MTA (`sendmail`, `postfix`, `exim`, etc.) installed and configured on the local machine
- Password-less SSH access from the local machine to each instrument computer (see below)
- Topspin running ICON-NMR on each remote instrument computer

## Installing

```sh
git clone https://github.com/greenwoodad/iconcheck
chmod +x ./iconcheck/iconcheck
```

## Getting Started

### 1. Set up sendmail

Check whether a sendmail-compatible binary is available:

```sh
which sendmail
```

If it is not installed, install it with your package manager:

```sh
sudo apt-get install sendmail   # Debian/Ubuntu
sudo yum install sendmail       # RHEL/CentOS
sudo dnf install sendmail       # Fedora/Rocky/Alma
```

then start the service with:

```sh
sudo systemctl start sendmail
sudo systemctl enable sendmail
```

The script defaults to `/usr/sbin/sendmail`. If your binary is elsewhere, update the `SendmailPath` variable near the top of the script. Common alternative locations are `/usr/bin/sendmail` and `/usr/lib/sendmail`.

To verify that sendmail is correctly configured (run as root or a privileged user):

```sh
sudo /usr/sbin/sendmail -bv your@email.address
```

Expected output: `your@email.address... deliverable: mailer esmtp, host mail.university.edu, user your@email.address`

### 2. Set up password-less SSH to instrument computers

Because iconcheck is intended to run as a cron job, it requires password-less SSH access from the local machine to each instrument computer. Full tutorials are available here:

- [How To Set Up Passwordless SSH Login](https://linuxize.com/post/how-to-setup-passwordless-ssh-login/)
- [OpenSSH Config File Examples](https://www.cyberciti.biz/faq/create-ssh-config-file-on-linux-unix/)

**Briefly:**

1. On the local machine, as the user who will run the cron job, generate an SSH key pair if you don't already have one:

```sh
ssh-keygen -t rsa -b 4096
```

Press Enter at the passphrase prompt to skip passphrase generation (required for unattended cron use).

2. Copy the public key to each instrument computer:

```sh
ssh-copy-id remote_username@remote_ip_address
```

If `ssh-copy-id` is not available:

```sh
cat ~/.ssh/id_rsa.pub | ssh remote_username@remote_ip_address \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

3. Add SSH aliases to `/etc/hosts` on the local machine:

```
198.51.100.50    av400.chem.university.edu    AV400
198.51.100.54    neo400.chem.university.edu   NEO400
198.51.100.59    hd500.chem.university.edu    HD500
```

The alias in the third column is what you will use as `SSHAlias` in the input file. After setup, you should be able to connect without a password:

```sh
ssh remote_username@AV400
```

Run this command manually for each aliased machine. It will prompt you to confirm the host fingerprint. After you do this once, connections should work automatically and the script can be run as a cron job.

### 3. Configure the input files

#### Main input file (`input/iconcheck_input`)

| Parameter | Description |
|---|---|
| `ScriptsPath` | Path to the iconcheck directory on the local machine. Use full path! |
| `ManagerEmail` | Email address of the NMR facility manager |
| `SSHAlias` | SSH alias for the instrument computer (must match `/etc/hosts`) |
| `RemoteUser` | Username for SSH login on the remote computer |
| `DebugPath` | Full path to the folder containing `IconDriverDebug` files on the remote computer (typically something like `/opt/topspin4.x.x/prog/curdir/nmr`) |
| `INMRPath` | Full path to the folder containing `Inmracct.brief` on the remote computer (typically something like `/opt/topspin4.x.x/conf/instr/spect/inmrusers`) |

> **Important:** Field values cannot contain spaces. Instrument lines can be commented out with `#`.

#### `input/error_table`

The error table maps ICON-NMR error patterns to email templates and delivery settings. Each row has four fields:

1. **Error string** — a pattern (supporting `.*` wildcards) that matches the error as it appears in `IconDriverDebug`. Special characters such as `"` should be escaped (e.g., `\"`).
2. **Email template** — filename of the email template in the `emailtxt/` folder to send to the user.
3. **Mail user?** — `y` or `n`
4. **Mail manager?** — `y` or `n`

The error string should match a unique pattern that appears in lines beginning with `Auto_SetAddHistoryItem` or `AutoSet_AddHistoryItem` in the `IconDriverDebug` file, and should not match unrelated lines.

When adding a new error, note that the script will send emails for any recent matching instances the first time it runs. To avoid this, temporarily set `mail user?` to `n` until those instances have been logged, then switch it back.

> **Important:** Separate fields with at least two spaces for correct parsing.

#### `input/addressbook`

Maps ICON-NMR usernames to email addresses. Each line contains a username and an email address separated by a space:

```
jsmith    jsmith@university.edu
mjones    mjones@university.edu
```

Users not listed in the addressbook will not receive emails. This file is independent of ICON-NMR's own user configuration.

## Usage

```sh
iconcheck [OPTIONS]... /path/to/iconcheck_input
```

| Option | Default | Description |
|---|---|---|
| `-h, -?, --help` | | Show help message |
| `-i, --input` | | Set input file (flag optional) |
| `-e, --email` | `y` | Set to `n` to skip sending emails |
| `-t, --trim` | `y` | Set to `n` to skip trimming `IconDriverDebug.Instrument.full` |
| `-v, --verbose` | | Print informational messages |

Default flag values and `SendmailPath` can be changed at the top of the script.

### Running as a cron job

Add an entry to your crontab (`crontab -e`):

```sh
*/5 * * * * /path/to/iconcheck "/path/to/input/iconcheck_input"
```

This runs iconcheck every 5 minutes. Cron will send mail only if the script produces output, which occurs when errors are detected or warnings are raised. No output means no mail.

## How it works

On each run, iconcheck:

1. SSHes into each instrument computer and rsyncs the latest `IconDriverDebug` and `Inmracct.brief` files
2. Appends new content to a persistent full-length debug log (`IconDriverDebug.Instrument.full`)
3. Scans the log for any error strings listed in `error_table` that have not been seen before
4. For each new error, looks up the associated user and spectrum name, then sends alert emails according to `error_table` settings
5. Logs processed errors to `debugerrors.Instrument.log` so they are not re-reported on subsequent runs

## Contributing

Pull requests and bug reports are welcome.

## Authors

- **Alex Greenwood** — [Greenwoodad](https://github.com/Greenwoodad)

## License

[MIT](https://choosealicense.com/licenses/mit/)


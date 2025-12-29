# repo-mail
Repossess your Google Workspace emails to a local Docker container.

Don't let your emails be held hostage! Repo them! Also works for Microsoft or whatever other cloud email providers.

## Benefits
Archive your emails locally and $top paying Google or Microsoft for old accounts. Protect against risk of loss of those accounts.

## Use Cases
Say an employee leaves and you want to stop paying monthly for their cloud account, but need to preserve the emails.

Or, what happens if the vendor (Google or Microsoft or whoever) decides they don't want you to use their system anymore? You would lose all your emails - what a disaster that would be.

# Instructions
1. Clone this repo.
2. Search and replace the example usernames with your own email addresses.
3. Synchronize the emails to your local computer.
4. Check that you have everything, then feel free to delete the account from your cloud provider.

# Detailed instructions and commands
### Run on your docker host
```bash
git clone git@github.com:thejohnha/repo-mail.git
cd repo-mail
```
### Set up the right folders for your account(s)
```bash
mkdir -p mail/username1             # adjust to alice if your email is alice@gmail.com
mkdir -p mail/username2             # adjust to bob if your email is bob@mydomain.com
chown -R 1000:1000 mail/username1   # alice
chown -R 1000:1000 mail/username2   # bob
chown -R 1000:1000 mail
chown -R 1000:1000 mbsync
mkdir -p roundcube/db
chown -R 33:33 roundcube/db         # Make it owned by UID 33 (www-data) which is the user roundcube runs as
```
### Update usernames and passwords in `dovecot/users`
```bash
nano dovecot/users
```
### Update usernames, passwords, configuration in `mbsync/.mbsyncrc`
```bash
nano mbsync/.mbsyncrc
```
This part requires the most tinkering to fit your situation. The examples are tuned for Google Workspace / Gmail. Adjust to accommodate your cloud provider. Please share (via pull request or other) examples for your cloud provider and I would be glad to update this repo!

There are 2 example account blocks in this file:
1. The first is for an account you want to keep using in the cloud, but you want a local backup/archive as insurance in case you lose the account.
2. Second scenario applies if the account belongs to a former employee: preserve the emails, then deactivate the account and discontinue billing.
3. *Feel free to delete any account block that doesn't apply to your situation.

Search for these keywords and replace them with your own account information:
```
- username1
- gmail.com
- username2
- mydomain.com
```

### PREVIEW: Use `mbsync` to ask server for list of all folders
This tests whether your config and app password works.
```bash
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc -l long-term-username1-gmail
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc -l one-time-username2
```

### Repo Your Mail!
#### Build & Start Infrastructure
```bash
docker compose build fetcher roundcube
docker compose up -d
```

### Run sync for `username2`
This runs the sync for the setup defined in the `.mbsyncrc` config file. This could take a while!
```bash
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc one-time-username2
#docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc long-term-username1-gmail
```
### Optional: Kill the sync
If you can't wait for it to finish, you can stop it with the `docker kill` command:
```bash
docker ps
docker kill <CONTAINER_ID>
```
Example: `docker kill 7d0faaa765ac`


# Verify Everything!
### VERIFY: Check size of mailboxes on disk
```bash
du -sh mail/*
```

### CHECK: Google Admin web console for size of gmail usage. Check against the size on disk and number of emails downloaded locally.
Replace `username2@mydomain.com` and `redacted_app_password` with your own credentials
```bash
## This creates a temporary container, installs openssl, and drops you into a shell
docker compose run --rm -u root --entrypoint /bin/sh fetcher
apk add openssl
## Connect to gmail
openssl s_client -connect imap.gmail.com:993 -crlf -quiet
a login username2@mydomain.com redacted_app_password
b STATUS "INBOX" (MESSAGES)
c STATUS "[Gmail]/All Mail" (MESSAGES)
d logout
```

### Log into Roundcube to verify everything is there.
```bash
http://localhost:8000         # (or whatever ip your docker host is on)
Username: username2
Password: redactedlocalpass   # this is the password that you defined in `dovecot/users`
```
#### Optional: Some of my preferred Roundcube settings:
```bash
Roundcube > Settings > Folders > flip on gmail folders
Roundcube > Settings > Preferences > Displaying Messages > Allow remote resources (images, styles) > always > Save
```

## OPTIONAL: Repeat the above for the "Long-Term" Sync (username1)
```bash
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc -l long-term-username1-gmail   # Optional Dry Run `-l switch`
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc long-term-username1-gmail      # Real Download
```


# Delete the cloud account - be careful!
Check all the above. Once satisfied, you can delete the Google Workspace or other cloud account.

Honestly I would wait a month or three to see if this is a stable solution for you. Try the backup and restore procedures (below) and make sure you're comfortable with everything. And only then delete the cloud account.


# Clean Up Local Account (After deleting the cloud account)
Once you are happy that your account(s) are safely in `dovecot` and you have deleted the Google Workspace or other cloud account:
```bash
nano mbsync/.mbsyncrc
```
and delete the entire "Account 2" or whatever section from `.mbsyncrc`. This prevents mbsync from trying to connect to a deleted account in the future.


# For Long-Run Sync Accounts:
You can set up a simple cronjob on your host machine to trigger the sync daily.

Adapt the folder path below to wherever you set up your `repo-mail` folder.

Add to crontab (This runs every day at 2:00 AM)
```bash
0 2 * * * cd /path/to/repo-mail && docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc long-term-username2-gmail
```
If you want to sync all the blocks in `.mbsyncrc` then use `-a` above instead of `-c /home/app/config/.mbsyncrc long-term-username2-gmail`


# Process the "next" account
Let's say an employee parts ways and you want to stop paying for their Google Workspace account, just edit the above files, search for all instances of `username2` and update to the next account you want to process.


# Backup
Backup each account into a separate `.tar.gz` file (along with all associated infrastructure files)
```bash
cd repo-mail
docker compose stop
tar -czvf    username1-gmail-backup-$(date +%F).tar.gz compose.yaml mbsync/ dovecot/ roundcube/ roundcube_build/ mail/username1/
tar -czvf username2-mydomain-backup-$(date +%F).tar.gz compose.yaml mbsync/ dovecot/ roundcube/ roundcube_build/ mail/username2/
docker compose up -d
```
## Safely move the `tar.gz` archives somewhere for storage

# Restore
Copy username2-mydomain-backup*.tar.gz to your new linux docker host.
```bash
mkdir -p repo-mail
tar -xzvf username2-mydomain-backup*.tar.gz -C repo-mail/
cd repo-mail
docker compose up -d
```
Open roundcube http://localhost:8000   # (or whatever ip your docker host is on)

User: username2

Pass: redactedlocalpass


# Possible Future Enhancements
- add command to get a count of all emails on cloud and count of all emails locally to help verify you got all the emails
- instructions to remove the Roundcube part of this solution if you don't need webmail and just want to use a desktop email client instead
- consider `imapsync` instead of `mbsync`
- upgrade `dovecot` to v2.4.x
- replace cron job with systemd timer
- create script to ask user for username, app password, sync options, and create the config files automatically

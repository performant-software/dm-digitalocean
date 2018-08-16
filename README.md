DM-Digital Ocean: Automated deploy scripts for creating an instance of DM 1.x on Digital Ocean.
============================================

Setting up your own DM instance:
-------------

The easiest way to host your own instance of DM for your projects is to use [Digital Ocean](https://www.digitalocean.com/) - a cloud computing platform where you can set up virtual server space for as little as $10/month. We’ve set up an optimized process to deploy DM to Digital Ocean.

The size of the “droplet” you will want to set up to host a DM instance will vary, but here is a basic estimate:

| Instance size                                       | Memory | SSD Disk |
| --------------------------------------------------- |:------:|:--------:|
| Small (1-5 individual projects, 30 GB total data)   | 2 GB   | 50 GB    |
| Medium (5-10 individual projects, 40 GB total data) | 2 GB   | 60 GB    |
| Large (10-20 individual projects, 60 GB total data) | 4 GB   | 80 GB    |

Alternatively, if you wish to set up a DM instance on your own server or another service, basic instructions are provided below.

Deploying your own instance to Digital Ocean
--------------

#### Creating an independent user authentication provider
DM supports login through the OAuth protocol, and by default connects to Google's and GitHub's authentication providers. In addition, an independent OAuth provider application can be quickly created using the accompanying [Simple OAuth2 provider](https://github.com/performant-software/oauth-provider) repository. To add this service, follow the deployment instructions in that repository's ReadMe, then navigate to your-provider-application-url/oauth/applications and add an entry for your DM instance. You should use 'DM' for the application name and add your-dm-application-url/accounts/oauth-callback/independent to the Redirect URI field (this field can accommodate a list of callback URIs separated by line breaks, which is useful if you wish to provide authentication to multiple DM instances). Click Submit to save the application configuration, and copy the Application Id (key) and Secret values shown on the subsequent page.

#### Setting configuration variables
In order to run the provisioning script to create an instance on Digital Ocean, you first need to create a local copy of this repository using `git clone` and fill in configuration values in the file machine-images/digitalocean.json. These include:
- `digital_ocean_api_token`: see the "How to Generate a Personal Access Token" in [Digital Ocean's API tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2). Your token must be provided here in order for the deployment process to connect with your Digital Ocean account.
- `superuser_id`: an identifier for a user who should have top-level admin status for all projects created in the instance. The format of this ID will be specific to the authentication provider this user will use; for the independent provider application, the ID will be "independent:username@example.com" with the user's email address replacing the latter part.
- `google_key` and `google_secret`: to enable login through Google's authentication service, see the [guide for Google's People API](https://developers.google.com/people/v1/getting-started) and fill in the key/secret values here after setting it up with your Google account.
- `github_key` and `github_secret`: to enable login through GitHub's authentication service, see the [guide for GitHub OAuth applications](https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/) and fill in the key/secret values after creating an application in your GitHub settings.
- `independent_provider_base_url`: the URL for the independent OAuth provider application you have deployed, including the protocol and the trailing slash character, as in "http://example.com/".
- `independent_provider_key` and `independent_provider_secret`: the Application ID and Secret values displayed by your independent OAuth provider application after adding your DM instance to its OAuth applications list.
- `independent_provider_description`: the name of your independent OAuth provider service as it will appear in the Login dropdown menu.

#### Installing required tools
You will need to install Ansible and Packer in order to run the provisioning script. See installation instructions at:
- http://docs.ansible.com/ansible/latest/intro_installation.html for Ansible (Mac OS X users may alternatively wish to use Homebrew)
- https://www.packer.io/intro/getting-started/install.html for Packer

#### Creating the Digital Ocean snapshot
After `cd`ing into your local copy of this repository, run the following two commands:

    $ ansible-galaxy install --role-file=provisioning/requirements.yml --roles-path=provisioning/roles
    $ packer build machine-images/digitalocean.json

The script will save a snapshot, which you can access through the web interface for your Digital Ocean account. To launch your DM instance, create a new droplet from this snapshot and keep the default selection for droplet sizing. When launched, the droplet will have run the DM application from the /home/dm directory.


Backups
-------
Backups of the RDF store are generated hourly and are automatically removed after 90 days in order to conserve storage space. To change the length of this window, edit the cron job for backup deletion with `crontab -e`.

To restore your DM instance from a backup file, you will need to shut the application down and relaunch it, passing the decompressed backup file as an argument.
- `ssh` as the root user into your instance server and navigate to /home/dm.
- Identify the backup file you want to restore (hourly backup files are located in dm-data/ttl-dumps). Run `cp your-backup-file-path /home/dm/backup-to-restore.ttl.gz`, replacing "your-backup-file-path" with the full path to the desired backup file, to copy it to a convenient location.
- In the /home/dm directory, run `gunzip backup-to-restore.ttl.gz` to decompress it.
- Run `ps -aux` to identify the process ID of the DM application. It will contain "java -jar dm-1.0-SNAPSHOT.jar" in the Command column. Run `kill` followed by the process ID to shut the application down.
- Run `nohup java -jar dm-1.0-SNAPSHOT.jar backup-to-restore.ttl > dm.log &` to launch the instance and restore it from the selected backup.

# st-mp

st-mp is the Stelligent Mini-Project.  Per the requirements, this will deploy an AWS EC2 instance that will host a static web site.

# Setting up the Environment

This is designed to deploy from an Ubuntu 14.04 LTS, which was used in development and testing.  A script was written to handle configuration of the environment that handles the provisioning and configuration process.

The basic steps are:
 1. Configure the local environment for Ansible
 2. Fetch the playbooks
 3. Launch the instance and deploy the website

## Deploying from the script

The script is designed to take a base Ubuntu 14.04 LTS system and install everything necessary to run the Ansible playbooks that deploy the EC2 instance and web site.  The script is available here:

```
wget https://s3-us-west-1.amazonaws.com/st-mp.provisioning/st-mp-deploy.sh
```

Or, if GIT is installed on the host, it can be cloned from:

```
https://github.com/Geryon/st-mp-deploy
```

The script has a few command line options, for the point of this documentation we'll just concern ourselves with two of those.  The script can be ran as follows:

```
bash st-mp-deploy.sh --hostname "<hostname>" --mail_to "<email_address>"
```

The command line options '--hostname' refers to the hostname (not FQDN) of the new instance.  This will simply populate the 'Name' tag in EC2, as well as any additional references to the system itself will be based on this.  There is also, '--mail_to', which is the e-mail address to mail the completion e-mail to.  When deployment is complete an e-mail is delivered to the address containing the public hostname with a link to the site and sanity check results.

All output from the script is sent to STDOUT and it is designed to stop running when any error is encountered.

Full documentation on this script can be found by passing it the '--man' option.  If no command line options are passed, defaults are used.  In this case the default hostname will be set to 'st-mp' and the default e-mail is 'nick@declario.com'.

Details on exactly what this script does are outlined in the 'Manual deployment' section below.

## Manual deployment

Manual deployment consists of configuring the system to fetch all dependencies to run Ansible, install git, and fetch our playbooks.  These are the necessary steps:

```
sudo apt-get update
sudo apt-get -y install git python-pip
sudo pip install boto pyyaml jinja2

git clone -b release1.8.2 https://github.com/ansible/ansible
cd ansible
git submodule update --init --recursive
cd -

git clone https://github.com/Geryon/st-mp
wget https://s3-us-west-1.amazonaws.com/st-mp.provisioning/secure_20150213.tar.gz -O - | tar zxf - -C st-mp/

export ANSIBLE_HOST_KEY_CHECKING=False
. ~/ansible/hacking/env-setup
```

At this point the local environement is configured to run Ansible.  Just a note, the 'secure_20150213.tar.gz' contains the AWS credentials as well as public and private keys.  I chose not to place it easily accessable in the repository.

The following command will deploy the actual system.  Replace 'hostname' and 'email' with your values.

```
ansible-playbook -i localhost, st-mp/playbooks/stelligent-mp.yml -e 'secure_dir="/home/nick/st-mp/secure" hostname="<hostname>" mail_to="<email>"' -t provision,deploy
```

As with the deployment script above, an e-mail will be delivered upon successful completion.

# Confirming the results

If everything completed successfully an e-mail will be sent out containing the public EC2 hostname.  Connecting via HTTP to this address will connect to the web server.  The server is configured to automatically redirect to HTTPS, which will cause a certificate verification error; please accept the certificate.  The Ansible playbooks create a self-signed cert for the server.  With the certificate accepted, the web page stating 'Automation for the People' will appear.  

At the end of the processes, just prior to the e-mail being sent, a sanity check is ran.  This sanity check will produce an HTML web page stored on the server it is ran on.  That page can be found by adding  '/sanity_check_results.html' to the URL.  A list of checks with their results will be displayed.  Successfull tests will display in green text, while errors will display in red.

Logging in via SSH can be accomplished using the 'stelligent-key' found in the 'secure/keys' directory and logging in as the stelligent user:

```
ssh -i ~/st-mp/secure/keys/stelligent-key stelligent@<ec2_hostname>
```

Note, for security the 'ubuntu' user has been removed from the system.

# Additional Notes

## DNS

There is a playbook for handling DNS via Route53.  If I had registered a domain to handle this, this playbook would automatically add the new EC2 instance to the domain via it's hostname.  For example, 'hostname.st-mp.com'.  Currently it is being ignored.

There are also two additional playbooks, 'sanitycheck.yml' and 'profile.yml'.  

## sanitycheck.yml

The 'sanitycheck.yml' playbook allows just the sanity check to be ran.  This requires one of two tags, 'remote' or 'local'.  When running in 'local' mode, this may be confusing, it does *not* run on the current server.  Instead it's refering to running locally on the specified server.  The server is specified via the inventory name.  For example, if we are attempting to connect to a server called 'test.st-mp.com', the command line would be as follows:

```
ansible-playbook -i test.st-mp.com, sanitycheck.yml -e 'secure_dir="/path/to/secure"' -t local
```

This will test all the internal connections and processes.  To confirm the site is accessable and responding correctly to external connections the script would be called like this:

```
ansible-playbook -i localhost, sanitycheck.yml -e 'secure_dir="/path/to/secure" hostname="test.st-mp.com"' -t remote
```

The later method will not create a status page but intsead if there is a failure, Ansible will report a failure of the playbook and the error returned will be the relevant problem the sanity check found.

## profile.yml

This playbook can be ran against an existing instance.  It's everything required to configure an EC2 instance.  Additionally, by limiting which tags are applied, it can be used to simply deploy an updated version of the website, as in this example:

```
ansible-playbook -i test.st-mp.com, profile.yml -e 'secure_dir="/path/to/secure"' -t deploy
```

This will confirm that everything is in place for the website and update the website from whatever the latest is in git.

## Sanity Check

The sanity check is written as an Ansible module called 'sanity_check'.  This is stored in the '~/st-mp/playbooks/library' directory.  Details on how to write a playbook that utilizes this module is fully outlined in the module itself.

# Repositories

Just a few notes regarding the repositories involved in this.  There are currently three repository, 'st-mp', 'st-mp-deploy' and 'st-mp-web'.  

'st-mp' is the main playbook repository, this contains everything required to launch and configure the host; however, it does *not* contain the contents of the website.

'st-mp-web' is the website.  The reason for placing this in a seperate repository is to mimic a production scenarion in which the website would be deployed from a git repository directly to the server.  The Ansible playbooks will treat it this way for deployment.

'st-mp-deploy' is the repository for the shell script that will create the environment.  See the second section of this document for full details or the script's man page.

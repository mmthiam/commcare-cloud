# Creating a new environment

In this example there's a root account for your AWS organization,
and the new environment will live in it's own linked account.

1. From the root account go to My Organization and create a new account.
    - use an email like `<mainaccount>+<env>@<org>`
2. Enter in new account email as if to log in and then go through the Forgot Password
    workflow to set password (to something randomized and strong). Save this password.
3. Log in using the new credentials.
4. Go to My Security Credentials and enable the root account's access key.
    Copy these to your `~/.aws/credentials` as
    ```
    [<account_alias>]
    aws_access_key_id = "..."
    aws_secret_access_key = "..."
    ```
5. Add `account_alias: <account_alias>` to `terraform.yml` of your env.
6. (If the S3 state bucket is under a different account) Go to https://console.aws.amazon.com/iam/home#/security_credential > Account Identifiers
    to get the Canonical User ID for the account, and then in an incognito tab log in
    under the account where the S3 state bucket lives, and under Permissions give
    your account access to the bucket using the Canonical User ID. 
7. Run `cchq <env> terraform init`
8. Run `cchq <env> terraform apply -target module.commcarehq.module.Users`
    and respond `yes` when prompted.
9. In AWS console, go to IAM users and click on your own username.
10. Create access key and copy them to `~/.aws/credentials`,
    replacing the root credentials you had there.
11. In AWS console under My Security Credentials,
    delete the access key you originally made for the root user.
12. Log out of AWS and then log back in as your IAM user. (To give the account a memorable
    account alias, make sure to set `account_alias` in `terraform.yml`.)


## Give access to team members

Terraform will create an IAM user account for each user specified in `meta.yml`.
The accounts are created without access keys or the ability to log in,
so they are essentially "inactive" accounts.

To "activate" an account:
1. Go to IAM users and select the user.
2. Click on the Security Credentials tab. Next to "Console password" it should say "Disabled";
    click on the "Manage password" link next to that.
3. Next to "Console access" click "Enable"
4. Check the box next to "Require password reset"
5. If the person is next to you, let them type in a password from your keyboard;
    otherwise take note of the autogenerated password and send it to them securely.
6. Send them a link to `https://<account_alias>.signin.aws.amazon.com/console`
    and have them log in and reset their password.
7. Have them to go to My Security Credentials to create an access key
    and write it in their `~/.aws/credentials` under `[<account_alias>]`.


## First terraform run

You can run terraform with
```
cchq <env> terraform apply
```

Because of a slight defect in our code, if you're using a site-to-site VPN
as configured through `vpn_connections` and `external_routes` in `terraform.yml`,
you have to hard-code VPN Gateway in `external_routes`. This means that first you have to
run terraform (to create the gateway), look up its id at
https://console.aws.amazon.com/vpc/home?#VpnGateways:sort=VpnGatewayId,
and edit `external_routes`  to
hardcode the gateway, and then run terraform again.

## VPN Setup

### Create the OpenVPN EC2 instance with Terraform
The first time you run `cchq <env> terraform apply`,
it will fail with a link to a terms of service you need to accept.
To do so:
1. First, make sure you are logged into https://console.aws.amazon.com/console/home
    _under the correct account_.
    (Each environment is it's own linked account,
    so be careful you're not logged into the
    e.g. staging account if you're doing this for production) 
2. Then click on the link in the output
3. Accept the terms and click "Continue to Subscribe".


### Gain temporary SSH access via your own IP
Once the VM is created, you still need to create a VPN user before you can use the VPN.
To do this, go into the console, find the vpn ec2 instance, go to its security group,
and click Inbound Traffic > Edit > Add Rule. Select Type "SSH" and Source "My IP",
and click Save.

Now you will be able to SSH into the VM.

To make a cert, you'll also need to open port 80, so click Add Rule again,
select Type HTTP, and Source "Anywhere", and click Save.

Finally, make sure to run

```bash
cchq <env> aws-fill-inventory
```

which will auto-generate an `[openvpn]` section to your inventory.ini.
For this to work, make sure you're using the inventory templating style. If you aren't,
you can just move `inventory.ini` to `inventory.ini.j2` before running that command,
and it'll generate `inventory.ini` for you. You can (can should) commit `inventory.ini`.

In order to log in from the public IP address, you'll need to uncomment the ansible_host
variable of `[openvpn]`. (Don't commit this change with the file!)

### Run the ovpn-init script

```
cchq <env> ssh openvpnas@openvpn
sudo ovpn-init --ec2
...
Please enter 'DELETE' to delete existing configuration:DELETE
...
Please enter 'yes' to indicate your agreement [no]: yes
... 
```
Make sure to type `yes` for the first prompt, and then just hit enter until it's done. 
Then set a password with
```
sudo passwd openvpn
```
This is the password you'll use to enter the admin web UI.

### Give others SSH access to the VPN machine
To give others SSH access to the VPN machine
(right now your access is because terraform created the VM with your public key)

```
cchq <env> bootstrap-users --limit openvpn -u openvpnas
cchq <env> deploy-stack --limit openvpn --skip-check
```

### Set up DNS and HTTPS cert

By whatever means you have, make a DNS entry that points a subdomain name
to the openvpn machine's public IP. The subdomain should be called `vpn.{{ SITE_HOST }}`,
e.g. if the site is at www.mycchqsite.org, it should be vpn.www.mycchqsite.org

Then run
```
cchq <env> ansible-playbook openvpn_playbooks/create_openvpn_cert.yml --skip-check -vvv
```

### Enable PAM in the web Admin UI

OpenVPN has a number authentication modes, and we're going to use
[PAM](https://docs.openvpn.net/command-line/authentication-options-and-command-line-configuration/#PAM_authentication),
which make VPN usernames and passwords mirror linux system user usernames and passwords.
In PAM authentication mode,
enabling a user just requires setting their linux user's password with `passwd`.

Go to `https://<vpn-subdomain-name>/admin` in your browser and log in with `openvpn`/`<password from above>`.
Then navigate to /admin/pam_configuration and click Use PAM.

### Activate your user

To activate a user, run

```
cchq <env> openvpn-activate-user <username>
```

and then have the user (in this case, yourself) change the password with

```
cchq <env> ssh openvpn -t passwd
```

providing first the ansible sudo user password, and then the new (secure!) password
as prompted.

### Connect to the VPN
Download the openvpn client and connect to the public IP with your username and password.

### Un-whitelist SSH traffic from your IP address
Finally once you've proven you can get on the VPN and log into VMs with their private IPs,
and once you've created a cert,
run `cchq <env> terraform apply` again to undo the temporary change you made via the console
that allowed you to SSH into the openvpn machine from the public internet,
and that allowed letsencrypt to make a request to port 80.

From here on out if you need to ssh into the VPN machine,
you can either manually whitelist yourself again, or else you'll have to connect to the VPN
and use the VPN machine's private IP address. Note that if you are using the private IP
and you run `sudo service openvpnas stop`, it will disconnect you from the VPN and you
won't be able to connect again. Then you will be forced to whitelist your IP
and use the public IP to ssh in and bring it back up. 

Finally, re-comment the ansible_host variable of `[openvpn]`
(or just `git checkout -- ...` this change).

### Make sure everything works

Now that you've turned off your special access, make sure you can
log on to the VPN again and then run

```
cchq <env> ssh openvpn
```

to make sure you can ssh onto the machine.

All done! Now to activate the other users, you can run the steps from "Activate your user"
above as users ask for access.

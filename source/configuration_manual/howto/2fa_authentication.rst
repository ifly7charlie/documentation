It's relatively easy to configure a push authentication service such as DUO Push Authentication with dovecot. If you also 
use dovecot as the authenticator for your SMTP server this will give you two factor authentication for both SMTP and IMAP/POP

This example is using the pam_duo module, and the Duo authentication app. Duo was chosen because it has a free tier with up to 10 free users

Be aware that client certifications are probably a better solution as you have no idea when the push will be sent

Steps:
* Install pam_duo
* Create DUO account and configure API access
* Configure the duo configuration file with API keys
* Create users
* Configure authentication & timeouts/caching

Install pam_duo & configure duo 
===============================
Follow instructions at https://duo.com/docs/duounix except configure the "dovecot" service rather than "ssh". 
The service name is in the args under passdb at the end of the line so you could use imap by changing it there

Create users
============
Users need to be configured in both the system password file and the DUO service. You should use the same username for both
and in DUO enable 2FA push

Configure Authentication
========================

In 10-auth.conf:

 disable_plaintext_auth = yes
 auth_cache_size = 1M
 auth_cache_ttl = 604800s
 auth_cache_verify_password_with_worker = yes
 auth_cache_negative_ttl = 10 minutes

I also have set to only use component before the @
 auth_username_format = %Ln

Uncomment the auth-system.conf.ext file 

In auth-system.conf.ext:
 passdb {
  driver = pam
  args = max_requests=1000 cache_key=%s%u dovecot
 }

%s%u is basically just %u as the service is hard coded to dovecot. You could use a service name of * to pick it based on how they are authenticating
and you can use a %r to include the remote IP so they need to re-authenticate if the IP address changes. I haven't done that as I don't want a new push
every time the mail account checks on a mobile network.

In 10-master.conf:
 service auth { 
   ....
   idle_kill=3600s
 }
This ensures that the auth master process doesn't die too often. The authentication cache is stored as a structure in this process so if the process
terminates then the cache is destroyed and a new 2FA push authentication will be needed. Without this the auth_cache_ttl is basically meaningless!

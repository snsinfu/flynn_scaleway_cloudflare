# Single-node Flynn on Scaleway behind Cloudflare

## 1. Create a server

Create an x86\_64 Ubuntu server on Scaleway. Add the server to your Ansible
inventory with `cluster_domain` variable set to the domain of your Flynn
cluster. Like this:

```ini
[flynn]
flynn.example.com ansible_host=10.10.10.10

[flynn:vars]
ansible_python_interpreter = /usr/bin/python3
ansible_port = 12345
ansible_user = deploy_user
ansible_ssh_private_key_file = ~/.ssh/id_rsa
cluster_domain = flynn.example.com
```

Customize ansible\_port and ansible\_user as you like.

## 2. Set up Cloudflare DNS

Go to your Cloudflare DNS dashboard and add these records:

    A      flynn    points to       10.10.10.10            ↗☁
    CNAME  *.flynn  is an alias of  flynn.example.com
    CNAME  app      is an alias of  app.flynn.example.com  -☁→

app.example.com will be accessible through Cloudflare.

## 3. Get an origin certificate

Go to your Cloudflare Crypto dashboard and generate an origin certificate for
*.flynn.example.com. Save the certificate as server.cert and the private key
as server.key in roles/flynn/files directory.

## 3. Init the server

Run

```
ansible-playbook init.yml
```

This will configure the server to:

- Use ansible\_port as ssh port
- Disallow root login
- Create a user named ansible\_user, which ssh login is allowed for

## 4. Setup flynn

Run

```
ansible-playbook setup.yml
```

This will:

- Set up a firewall
- Add a swap file
- Install ZFS
- Install Docker-CE
- Install single-node Flynn
- Retrieve controller key into flynn-controller.key file

## 5. Connect to Flynn

Calculate the certificate pin:

```
CERT_PIN=$(openssl x509 -inform PEM -outform DER < roles/flynn/files/server.cert | openssl dgst -binary -sha256 | openssl base64)
```

Then do this:

```
flynn cluster add -p ${CERT_PIN} default flynn.example.com $(cat flynn-controller.key)
```

## 6. Publish an app

Create a Flynn app and add a route:

```
flynn add route http app.example.com
```

Then your app will be accessible via Cloudflare at https://app.example.com/.

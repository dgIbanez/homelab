# Manual Installation of step-ca

This guide has been tested on Bash in Linux environments.

## 1. Pull down the Docker image
Get the latest version of `step-ca`:

```bash
docker pull smallstep/step-ca
```

## 2. Bring up PKI bootstrapping container
The Docker volume `step` will hold your CA configuration, keys, and database.

```bash
docker run -it -v step:/home/step smallstep/step-ca step ca init --remote-management --acme
```

The `init` command will guide you through the bootstrapping process. Example output:

```plaintext
✔ What would you like to name your new PKI? (e.g. Smallstep): Smallstep
✔ What DNS names or IP addresses would you like to add to your new CA? (e.g. ca.smallstep.com[,1.1.1.1,etc.]): localhost
✔ What address will your new CA listen at? (e.g. :443): :9000
✔ What would you like to name the first provisioner for your new CA? (e.g. you@smallstep.com): admin@smallstep.com
✔ What do you want your password to be? [leave empty and we'll generate one]:

Generating root certificate... done!
Generating intermediate certificate... done!

✔ Root certificate: /home/step/certs/root_ca.crt
✔ Root private key: /home/step/secrets/root_ca_key
✔ Root fingerprint: fa08cceda8501b1d93d275cfc614a5af2a37c6c72e674192b4598808c5bae91e
✔ Intermediate certificate: /home/step/certs/intermediate_ca.crt
✔ Intermediate private key: /home/step/secrets/intermediate_ca_key
✔ Database folder: /home/step/db
✔ Default configuration: /home/step/config/defaults.json
✔ Certificate Authority configuration: /home/step/config/ca.json
✔ Admin provisioner: admin@smallstep.com (JWK)
✔ Super admin subject: step

Your PKI is ready to go. To generate certificates for individual services see 'step help ca'.
```

**Important**: Save the root fingerprint value! You'll need it for client bootstrapping.

## 3. Place the PKI password in a known safe location
The image expects the intermediate CA private key password to be placed in `/home/step/secrets/password`. Bring up the shell prompt in the container and write that file:

```bash
docker run -it -v step:/home/step smallstep/step-ca sh
```

Inside your container, write the password file into the expected location:

```bash
echo -n "<your password here>" > /home/step/secrets/password
```

Your CA is now configured and ready to run.

## 4. Start step-ca
The CA runs an HTTPS API on port 9000 inside the container. Expose the server address locally and run the step-ca with:

```bash
docker run -d -p 9000:9000 -v step:/home/step smallstep/step-ca
```

Your local step CLI is now configured to use the container instance of `step-ca` and the new root certificate is trusted by the host environment.

## 5. Run a health check

Verify that your CA is running:

```bash
curl https://localhost:9000/health
```

**Output:**

```json
{"status":"ok"}
```

## 6. Copy the Root Certificate to Other Hosts

To ensure other systems trust the CA, copy the root certificate from the step-ca container to your target LXC instances. For example:

```bash
docker cp <container-id>:/home/step/certs/root_ca.crt .
scp ./root_ca.crt user@<LXC_ip>:/tmp/
```
On the target host (e.g., Debian 12 LXC), move the certificate to the trusted CA directory:
```bash
mv /tmp/root_ca.crt /usr/local/share/ca-certificates/
update-ca-certificates
```
This loads the root CA into the trusted certificate pool located at /etc/ssl/certs/root_ca.pem. The system is now ready to trust certificates issued by your step-ca.

## 7. Solicitar certificado
```bash
certbot certonly --standalone -d <lxc>.homelab.local --server https://step-ca.homelab.local:9000/acme/acme/directory --agree-tos --email <tu mail aqui> -v
```

## 8. Configurar renovación

Para configurar la renovación automática de los certificados, asegúrate de que el servidor de la CA esté especificado correctamente en el archivo de configuración `cli.ini` de Certbot. Para ello, agrega la siguiente línea al archivo:

```ini
server = https://step-ca.homelab.local:9000/acme/acme/directory
```

Esto asegura que Certbot utilice el servidor ACME de tu `step-ca` para las renovaciones.


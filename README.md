```bash
sudo -s
export HOST=my.domain
mkdir -p /root/.docker && cd /root/.docker

# generate CA private and public keys
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -subj "/CN=$HOST" -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# create a server key and certificate signing request (CSR). 
# Make sure that “Common Name” matches the hostname you use to connect to Docker
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr

# sign the public key with our CA
# Since TLS connections can be made through IP address as well as DNS name, 
# the IP addresses can also be specified when creating the certificate
echo subjectAltName = DNS:$HOST,IP:127.0.0.1 >> extfile.cnf

# Set the Docker daemon key’s extended usage attributes to be used only 
# for server authentication
echo extendedKeyUsage = serverAuth >> extfile.cnf

# Now, generate the signed certificate
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf

# For client authentication, create a client key and certificate signing request
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr

# To make the key suitable for client authentication, create a new extensions config file
echo extendedKeyUsage = clientAuth > extfile-client.cnf

# Now, generate the signed certificate
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile-client.cnf

# Cleanup
rm -v client.csr server.csr extfile.cnf extfile-client.cnf
chmod -v 0400 ca-key.pem key.pem server-key.pem
chmod -v 0444 ca.pem server-cert.pem cert.pem

# Secure by default
exit
mkdir -pv ~/.docker
sudo cp -v /root/.docker/{ca,cert,key}.pem ~/.docker
sudo chown -R $USER:$USER ~/.docker
```

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/override.conf
```

```bash
# file: "/etc/systemd/system/docker.service.d/override.conf"
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376 --tlsverify --tlscacert /root/.docker/ca.pem --tlscert /root/.docker/server-cert.pem --tlskey /root/.docker/server-key.pem -H unix:///var/run/docker.sock
```

Now reload and restart.

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```

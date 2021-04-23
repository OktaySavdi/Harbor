![image](https://user-images.githubusercontent.com/3519706/115870945-9b989200-a448-11eb-9815-804ae3e0e142.png)

Harbor is a docker container image registry which is open source and it can be self hosted. I needed to install a docker registry in Centos 7 machine recently. So I thought to document the process I have followed.

Before going in to installation details, why Harbor? The reasons for us to choose harbor was its image scanning capabilities, helm chart repository capability and it can be freely used as we want. There are some reasons.

If there are any other installation for docker exist in the server, we need to first uninstall them.
```bash
sudo yum -y remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
Then we have to setup our repository.
```bash
sudo yum install -y yum-utils 

sudo yum-config-manager \
    --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
To get the latest stable release, we can run below command.
```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io
```

Then we have finished installing docker engine. We have to start it below commands.
```bash
sudo systemctl start docker
```
## Install Docker Compose

Docker Compose Release - [enter link description here](https://github.com/docker/compose/releases)

Next prerequisite is Docker compose. The below command gives the latest docker compose and if you want to change the version, just use the relevant version number in the command.
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
**_## Giving permissions_**  
```bash
sudo chmod +x /usr/local/bin/docker-compose
```
**_## Test installation_**  
```bash
docker-compose --version
```
And other than above two requirements, you need to have openSSL in your machine.

## Harbor installer

Now we have completed all the prerequisites and then we can download the Harbor installer to install harbor. There are two variants of installers. Online and offline. I was using online installer.

**_## Change the version accordingly_**  
Harbor Release - [URL](https://github.com/goharbor/harbor/releases)
```bash
wget https://github.com/goharbor/harbor/releases/download/v2.2.1/harbor-online-installer-v2.2.1.tgz
```
## Extract the downloaded file_** 
```bash
tar xvf harbor-online-installer-v2.2.1.tgz
```
Now we have got the installer. Then we’ll add the HTTPS access to the repository. If this is going in production level, we will need a certified authority to sign the certificate. Here I’m generating my own CA.

## Generate Certificates

**1. Generate CA private key**  
```bash
openssl genrsa -out ca.key 4096
```
**2. Generate CA certificate (Change the values accordingly)**  
```bash
openssl req -x509 -new -nodes -sha512 -days 3650 \
-subj "/C=CN/ST=Colombo/L=Colombo/O=Organization/OU=Personal/CN=harbor-registry.com" \
-key ca.key \
-out ca.crt
```
**3. Generate server certificate(Change the values accordingly)**  
```bash
openssl genrsa -out harbor-registry.com.key 4096
```
**4. Generate certificate signing request(Change the values accordingly)**  
```bash
openssl req -sha512 -new \
-subj "/C=CN/ST=Colombo/L=Colombo/O=Organization/OU=Personal/CN=harbor-registry.com" \
-key harbor-registry.com.key \
-out harbor-registry.com.csr
```
**5. Generate an x509 v3 extension file.(Change the values accordingly)**  
```bash
cat > v3.ext <<-EOF  
authorityKeyIdentifier=keyid,issuer  
basicConstraints=CA:FALSE  
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment  
extendedKeyUsage = serverAuth  
subjectAltName = @alt_names  
  
[alt_names]  
DNS.1=harbor-registry.com  
DNS.2=harbor-registry  
DNS.3=host-name  
EOF
```
**6. Use above file to generate certificate.(Change the values accordingly)**  
```bash
openssl x509 -req -sha512 -days 3650 \
-extfile v3.ext \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-in harbor-registry.com.csr \
-out harbor-registry.com.crt
```
**7. Provide the certificates for Harbor.**  
```bash
mkdir -p /data/cert/ 
cp harbor-registry.com.crt /data/cert/
cp harbor-registry.com.key /data/cert/
```
**8. For docker to use this cert we need to convert .crt to .cert. Then we need to move them to the appropriate folder.**  
```bash
openssl x509 -inform PEM -in harbor-registry.com.crt -out harbor-registry.com.certcp harbor-registry.com.cert /etc/docker/certs.d/harbor-registry.com/

mkdir /etc/docker/certs.d/harbor-registry.com/
cp harbor-registry.com.key /etc/docker/certs.d/harbor-registry.com/
cp ca.crt /etc/docker/certs.d/harbor-registry.com/
```
**9. Restart docker**  
```bash
systemctl restart docker
```
## Deploy Harbor

To deploy Harbor, first we need to do some configuration changes in harbor.yml file. In the extracted harbor directory, you’ll find a file named harbor.yml.tmpl. We have to first take a copy of that file and create our own yml file.
```bash
cp harbor.yml.tmpl harbor.yml
```
**_## Inside this file change the below parts accordingly._**
```bash
hostname: harbor-registry  
....  
# https related config  
https:  
  # https port for harbor, default is 443  
  port: 443  
  # The path of cert and key files for nginx  
  certificate: /data/cert/harbor-registry.com.crt  
  private_key: /data/cert/harbor-registry.com.key

Note : And there are many other configurations you can change if you want.
```
## Access harbor

To push and pull to the registry from other machines, we need one more step. We have to add this registry to insecure registries.
```bash
vim /etc/docker/daemon.json  
{
 "insecure-registries": [
      "harbor-registry.com"
 ]
}
```
**Add registery name on hosts file**
```bash
cat /etc/hosts
```
```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 harbor-registry.com harbor-registry
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```
Inside that harbor directory you can see an executable named prepare. Then we’ll run it.

-   `./prepare`

For first time installation you don’t need to do the down step. Then we have to up the service.

-   `docker-compose down -v`
-   `docker-compose up -d`

Then we can do the docker login.

`user: admin`
`password: Harbor12345`
```bash
docker login harbor-registry.com
```
**_## To test, we'll just get a image_**
```bash
docker pull busybox
```
**_## Then to push it into our registry,_**  
```bash
docker tag busybox:latest harbor-registry.com/myExampleProject/busybox:latest
```
Now you can go to the browser and check the image in the repository.

To pull the docker image,
```bash
docker push harbor-registry.com/myExampleProject/busybox:latest
```

**#Harbor Console**

user: admin
password: Harbor12345

![image](https://user-images.githubusercontent.com/3519706/115871211-ed411c80-a448-11eb-9fb6-054646881648.png)

![image](https://user-images.githubusercontent.com/3519706/115871487-4b6dff80-a449-11eb-93f6-0c85b3459a23.png)

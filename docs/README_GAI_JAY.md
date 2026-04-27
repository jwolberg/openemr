# Clinical Co-Pilot — OpenEMR Local Setup Notes

## Project Context

This project uses OpenEMR as the foundation codebase for the Clinical Co-Pilot project. OpenEMR is a large, real-world, open-source Electronic Health Record and medical practice management system. The goal is not to build a clinical application from scratch, but to understand and extend an existing healthcare application by integrating an AI agent into the existing system.

Fork source:
https://github.com/openemr/openemr

Into:
https://github.com/jwolberg/openemr

## Stage 1 — Run OpenEMR Locally
### Environment
MacBook Pro
macOS
Docker Desktop
Docker Compose

### Docker
Docker version 29.4.1, build 055a478
Docker Compose version v5.1.3

### Start OpenEMR
cd ~/workspace/openemr/docker/development-easy
docker compose up

#### This starts services
OpenEMR HTTP:   http://localhost:8300/
OpenEMR HTTPS:  https://localhost:9300/
phpMyAdmin:     http://localhost:8310/
Mailpit:        http://localhost:8025/
CouchDB:        http://localhost:5984/
MySQL:          localhost:8320

The working local app is available at:
https://localhost:9300/  ( or :8300)
Login: admin / pass


### Stop OpenEMR
docker compose down

### Check services
docker compose ps

### logs
docker compose logs --tail=120 openemr
docker compose logs --tail=120 mysql

## Sample data check

After starting OpenEMR locally, I checked whether the Docker development setup included existing sample patients.

Database query used:
SELECT COUNT(*) AS patient_count FROM patient_data;
 docker compose exec mysql mariadb -uopenemr -popenemr openemr -e "SELECT COUNT(*) AS patient_count FROM patient_data;"

patient_count == 0

 ### sample patients need to be created/imported for audit and demo workflows


## Demo deployment
Google Compute Engine VM, install Docker, and run OpenEMR using the official production Docker Compose pattern. 
OpenEMR’s official Docker image supports automated installation/configuration and requires a companion MySQL/MariaDB container. 
OpenEMR’s own installation guide also points to the official Docker image as the modern plug-and-play Docker option


My OpenEMR is deployed on a Google Compute Engine VM using Docker Compose.

- OpenEMR runs in a Docker container.
- MariaDB runs in a Docker container.
- Caddy provides HTTPS reverse proxy.
- The demo uses an sslip.io hostname mapped to the VM static IP.
- Persistent Docker data is stored on a GCE persistent disk.
- This environment is for synthetic patient data only.

Demo URL:

https://openemr.136-118-242-198.sslip.io

## Data policy
This environment is for synthetic patient data only.
This is not a HIPAA-ready production deployment.

### Shape
- Google Compute Engine VM
- Docker and Docker Compose
- OpenEMR container
- MariaDB container
- Persistent Docker volumes / persistent disk
- HTTPS reverse proxy
- Synthetic patient data only

#### gcloud init
##### 1. env vars
export PROJECT_ID="clinic-copilot-1"
export REGION="us-west1"
export ZONE="us-west1-a"
export VM_NAME="openemr-demo"
export DATA_DISK_NAME="openemr-data-disk"

gcloud config set project $PROJECT_ID

gcloud services enable compute.googleapis.com

gcloud compute addresses create openemr-ip \
  --region=$REGION

gcloud compute addresses describe openemr-ip \
  --region=$REGION \
  --format="get(address)"

 136.118.242.198

that IP can point to demo domain to it, for example:
  openemr.yourdomain.com  A  <STATIC_IP> 

  or  use 
  openemr.136-118-242-198.sslip.io


##### create the VM
gcloud compute instances create $VM_NAME \
  --zone=$ZONE \
  --machine-type=e2-standard-2 \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=50GB \
  --address=openemr-ip \
  --tags=http-server,https-server

##### Open firewall ports 80 and 443:
  gcloud compute firewall-rules create allow-openemr-http \
  --allow=tcp:80 \
  --target-tags=http-server \
  --description="Allow HTTP for Caddy certificate issuance"

gcloud compute firewall-rules create allow-openemr-https \
  --allow=tcp:443 \
  --target-tags=https-server \
  --description="Allow HTTPS for OpenEMR demo"

  gcloud compute disks create $DATA_DISK_NAME \
  --zone=$ZONE \
  --size=100GB \
  --type=pd-balanced

gcloud compute instances attach-disk $VM_NAME \
  --zone=$ZONE \
  --disk=$DATA_DISK_NAME

  ##### SSH into the VM:
gcloud compute ssh $VM_NAME --zone=$ZONE

##### Find the attached disk:

lsblk

##### You will likely see something like /dev/sdb. Format and mount it:

sudo mkfs.ext4 -F /dev/sdb
sudo mkdir -p /srv/openemr
sudo mount /dev/sdb /srv/openemr

##### Persist the mount across reboots:

sudo blkid /dev/sdb


=>> /dev/sdb: UUID="6ad8faed-ca1b-4476-9c0d-6db917186ec9" BLOCK_SIZE="4096" TYPE="ext4"

#####  Copy the UUID, then:

sudo nano /etc/fstab

add line: 
UUID=6ad8faed-ca1b-4476-9c0d-6db917186ec9 /srv/openemr ext4 defaults,nofail 0 2


3. Install Docker and Docker Compose

Docker’s current Ubuntu docs support Ubuntu 24.04 and install Docker Engine plus the Compose plugin from Docker’s apt repository.

sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Let your user run Docker:

sudo usermod -aG docker $USER
exit


###### SSH  in:

gcloud compute ssh $VM_NAME --zone=$ZONE

Verify:

docker --version
docker compose version



4. Create the OpenEMR deployment directory
sudo mkdir -p /srv/openemr/app
sudo chown -R $USER:$USER /srv/openemr

cd /srv/openemr/app

mkdir -p data/mariadb
mkdir -p data/openemr-sites
mkdir -p data/openemr-logs
mkdir -p data/caddy-data
mkdir -p data/caddy-config
5. Create .env
nano .env

``` DOMAIN=openemr.yourdomain.com   # domain not in use. use IP address ```

DOMAIN=openemr.136-118-242-198.sslip.io
MYSQL_ROOT_PASSWORD=root_password
MYSQL_DATABASE=openemr
MYSQL_USER=openemr
MYSQL_PASSWORD=root_password

OE_USER=admin
OE_PASS=pass


6. Create docker-compose.yml
nano docker-compose.yml

services:
  mariadb:
    image: mariadb:11.8
    container_name: openemr-mariadb
    restart: unless-stopped
    command: ["mariadbd", "--character-set-server=utf8mb4"]
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - databasevolume:/var/lib/mysql
    networks:
      - openemr-net
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--su-mysql", "--connect", "--innodb_initialized"]
      start_period: 60s
      interval: 30s
      timeout: 5s
      retries: 5

  openemr:
    image: openemr/openemr:latest
    container_name: openemr-app
    restart: unless-stopped
    depends_on:
      mariadb:
        condition: service_healthy
    environment:
      MYSQL_HOST: mariadb
      MYSQL_ROOT_PASS: ${MYSQL_ROOT_PASSWORD}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASS: ${MYSQL_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      OE_USER: ${OE_USER}
      OE_PASS: ${OE_PASS}
    volumes:
      - logvolume01:/var/log
      - sitevolume:/var/www/localhost/htdocs/openemr/sites
    networks:
      - openemr-net
    expose:
      - "80"

  caddy:
    image: caddy:2
    container_name: openemr-caddy
    restart: unless-stopped
    depends_on:
      - openemr
    ports:
      - "80:80"
      - "443:443"
    environment:
      DOMAIN: ${DOMAIN}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data/caddy-data:/data
      - ./data/caddy-config:/config
    networks:
      - openemr-net

networks:
  openemr-net:
    driver: bridge

volumes:
  databasevolume:
  sitevolume:
  logvolume01:


This intentionally does not expose OpenEMR directly to the host. Only Caddy gets public ports 80 and 443.

7. Create the Caddy reverse proxy config

Caddy automatically provisions and renews HTTPS certificates and redirects HTTP to HTTPS by default.

nano Caddyfile

Paste:

{$DOMAIN} {
    reverse_proxy openemr:80
}


8. Start the stack
docker compose pull
docker compose up -d


##### Check status:

docker compose ps

Watch logs:

docker compose logs -f caddy
docker compose logs -f openemr
docker compose logs -f mariadb

##### Once Caddy gets its certificate, visit:

https://openemr.yourdomain.com
https://openemr.136-118-242-198.sslip.io/ 

Login with:

Username: value of OE_USER
Password: value of OE_PASS




validation steps

cd /srv/openemr/app
docker compose ps

### test persistence:

docker compose restart
  --> Log back in and confirm OpenEMR still loads.

Create test patient:

Name: Alex Fake
DOB: 2000-01-01

restart again -- > docker compose restart
--> Confirm the patient is still there.

##### Useful maintenance commands
cd /srv/openemr/app

docker compose ps
docker compose logs -f openemr
docker compose logs -f caddy
docker compose logs -f mariadb
docker compose restart



##### Backup database:

source .env

mkdir -p /srv/openemr/backups

docker compose exec mariadb mariadb-dump \
  -u root \
  -p${MYSQL_ROOT_PASSWORD} \
  openemr > /srv/openemr/backups/openemr-db-$(date +%Y%m%d-%H%M%S).sql




#### update image
Google Artifact Registry is the right place to push images for GCP. Image paths look like us-west1-docker.pkg.dev/PROJECT_ID/REPO_NAME/IMAGE_NAME:TAG, and Docker Compose will recreate containers when the image or config changes while preserving mounted volumes.

1. Create an Artifact Registry repo

From your local machine:

export PROJECT_ID="your-gcp-project-id"
export REGION="us-west1"
export REPO="openemr-images"

gcloud config set project $PROJECT_ID

gcloud services enable artifactregistry.googleapis.com

gcloud artifacts repositories create $REPO \
  --repository-format=docker \
  --location=$REGION \
  --description="Docker images for OpenEMR demo"

Configure Docker auth:

gcloud auth configure-docker ${REGION}-docker.pkg.dev

Artifact Registry requires the repo to exist before pushing, and the pushing account needs Artifact Registry Writer access. The VM or runtime account pulling the image needs Artifact Registry Reader access.

2. Build and push the image

From your codebase:

export TAG=$(git rev-parse --short HEAD)
export IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/openemr:${TAG}"

docker build -t $IMAGE .
docker push $IMAGE

Example image:

us-west1-docker.pkg.dev/my-project/openemr-images/openemr:a1b2c3d

If you are just updating to a different official OpenEMR image, you can skip building/pushing and simply change the image tag in docker-compose.yml, for example:

image: openemr/openemr:7.0.4.0

to:

image: openemr/openemr:8.0.0.3

But for your own modified image, use Artifact Registry.

3. Give the VM permission to pull

Find the VM service account:

gcloud compute instances describe openemr-demo \
  --zone=us-west1-a \
  --format="get(serviceAccounts.email)"

Grant Artifact Registry Reader:

export VM_SERVICE_ACCOUNT="your-vm-service-account@your-project.iam.gserviceaccount.com"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${VM_SERVICE_ACCOUNT}" \
  --role="roles/artifactregistry.reader"

SSH into the VM and configure Docker auth:

gcloud compute ssh openemr-demo --zone=us-west1-a

gcloud auth configure-docker us-west1-docker.pkg.dev
4. Backup before deploying

On the VM:

cd /srv/openemr/app
source .env

mkdir -p /srv/openemr/backups

docker compose exec mariadb mariadb-dump \
  -u root \
  -p"$MYSQL_ROOT_PASSWORD" \
  "$MYSQL_DATABASE" > /srv/openemr/backups/pre-deploy-$(date +%Y%m%d-%H%M%S).sql

For your synthetic-only demo this is enough.

5. Update docker-compose.yml

On the VM:

cd /srv/openemr/app
nano docker-compose.yml

Change the openemr image from:

image: openemr/openemr:7.0.4.0

to your pushed image:

image: us-west1-docker.pkg.dev/YOUR_PROJECT_ID/openemr-images/openemr:a1b2c3d

Then deploy:

docker compose pull openemr
docker compose up -d openemr
docker compose ps

docker compose pull pulls service images, and docker compose up -d starts/recreates the changed service in detached mode.

6. Verify
docker compose ps
docker compose logs --tail=100 openemr

Test internally:

docker exec openemr-app curl -i http://localhost

Expected:

HTTP/1.1 302 Found
Location: interface/login/login.php?site=default

Test public URL:

source .env
curl -Ik https://$DOMAIN

Then open:

https://openemr.136-118-242-198.sslip.io


7. Rollback

Before deploying, write down the previous image:

grep "image:" docker-compose.yml

Rollback is just changing the image tag back:

image: openemr/openemr:7.0.4.0

or to the prior Artifact Registry tag:

image: us-west1-docker.pkg.dev/YOUR_PROJECT_ID/openemr-images/openemr:previous-good-sha

Then:

docker compose pull openemr
docker compose up -d openemr
docker compose logs --tail=100 openemr
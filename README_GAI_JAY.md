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

# A Deux Pas - CD Pipeline

Continuous Deployment pipeline for A Deux Pas frontend (Angular) and backend (Spring Boot) applications.

## Overview

This pipeline uses Jenkins with Ansible to deploy applications to DEV and PROD environments.

## Architecture

- **Jenkins**: Orchestrates the deployment pipeline
- **Ansible**: Executes deployment tasks on target hosts
- **Nexus**: Artifact repository (source for JAR and tar.gz files)

## Environments

Both DEV and PROD environments run on the same host with different directories and ports.

### Common Configuration
- **Host**: magnolia.readresolve.tech:50000
- **User**: magnolia (regular user, no sudo required)
- **Auth**: Basic auth (user=webuser)

### DEV Environment
- **Frontend**: `/srv/readresolve.tech/magnolia/www/a-deux-pas-dev`
- **Backend**: `/srv/readresolve.tech/magnolia/api/a-deux-pas-dev`
- **Backend Port**: 8081
- **Frontend URL**: https://magnolia.readresolve.tech/a-deux-pas-dev
- **Backend URL**: https://magnolia.readresolve.tech/api/a-deux-pas-dev

### PROD Environment
- **Frontend**: `/srv/readresolve.tech/magnolia/www/a-deux-pas-prod`
- **Backend**: `/srv/readresolve.tech/magnolia/api/a-deux-pas-prod`
- **Backend Port**: 8080
- **Frontend URL**: https://magnolia.readresolve.tech/a-deux-pas-prod
- **Backend URL**: https://magnolia.readresolve.tech/api/a-deux-pas-prod

## Jenkins Parameters

The pipeline accepts three parameters:

1. **BACK_APP_VERSION** (String): Backend version to deploy (leave empty to skip)
2. **FRONT_APP_VERSION** (String): Frontend version to deploy (leave empty to skip)
3. **TARGET_ENV** (Choice): DEV or PROD

## Required Jenkins Credentials

1. **nexus-credentials** (Username/Password): Nexus repository credentials
2. **magnolia-ssh-password** (Secret text): SSH password for magnolia user
3. **webuser-http-password** (Secret text): HTTP basic auth password for version verification

## Deployment Flow

1. **Validation**: Check that at least one version is specified
2. **Environment Preparation**: Load SSH and HTTP credentials from Jenkins
3. **Backend Deployment** (if BACK_APP_VERSION provided):
   - Stop existing application (systemd user service)
   - Clean deployment directory
   - Download JAR from Nexus
   - Copy environment-specific application.properties
   - Create/update systemd user service
   - Start application with correct port (8080 for PROD, 8081 for DEV)
   - Verify version via HTTP
4. **Frontend Deployment** (if FRONT_APP_VERSION provided):
   - Clean deployment directory
   - Download tar.gz from Nexus
   - Extract to deployment directory
   - Verify version via HTTP

## Version Verification

Both frontend and backend include version JSON files that are verified after deployment:
- **Frontend**: `/frontend-version.json`
- **Backend**: `/backend-version.json`

The pipeline checks that the deployed version matches the requested version.

## Usage Examples

### Deploy both apps to DEV
```groovy
BACK_APP_VERSION: "1.0.0-SNAPSHOT"
FRONT_APP_VERSION: "1.0.0-SNAPSHOT"
TARGET_ENV: DEV
```

### Deploy only backend to PROD
```groovy
BACK_APP_VERSION: "1.0.0"
FRONT_APP_VERSION: ""
TARGET_ENV: PROD
```

### Deploy only frontend to PROD
```groovy
BACK_APP_VERSION: ""
FRONT_APP_VERSION: "1.0.0"
TARGET_ENV: PROD
```

## Directory Structure

```
deploy-a-deux-pas/
├── Jenkinsfile                    # Main CD pipeline
├── ansible/
│   ├── inventory.ini              # Ansible inventory (DEV and PROD)
│   ├── deploy.yml                 # Main deployment playbook
│   └── roles/
│       ├── deploy-backend/        # Backend deployment role
│       │   ├── defaults/main.yml
│       │   ├── tasks/main.yml
│       │   └── templates/
│       │       ├── application.properties.j2
│       │       └── backend.service.j2
│       └── deploy-frontend/       # Frontend deployment role
│           ├── defaults/main.yml
│           └── tasks/main.yml
└── README.md
```

## Notes

- Both DEV and PROD use the same host (magnolia.readresolve.tech) with different directories
- Backend is deployed first when both are specified (as per requirements)
- All deployment directories are checked for write access before deployment
- Both DEV and PROD backends run as systemd user services (no sudo required)
- Different ports are used: DEV (8081), PROD (8080)
- Each environment has separate systemd service: backend-adp-dev, backend-adp-prod
- Failed deployments are clearly reported with error messages

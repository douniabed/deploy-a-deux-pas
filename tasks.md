we want to develop CD pipeline to deploy a-deux-pas front (angular) and back (java springboot) apps using jenkinsfile and ansible role
FRont CI : /home/donia/Dev/a-deux-pas-fork/front-a-deux-pas/pipeline/Jenkinsfile
Back CI : /home/donia/Dev/a-deux-pas-fork/back-a-deux-pas/pipeline/Jenkinsfile

Jenkins will use ansible agent label
ansible will use an inventory to reference 2 envs DEV and PROD each env will reference 2 hots one for front other for back, even if they refer to the same actual host machine

===========
PROD
machine for front and back
ssh magnolia@magnolia.readresolve.tech -p 50000
ssh password => stored in jenkins credential id magnolia-ssh-password (Secret text)

Front deployment dir
magnolia@vps-3229ca35:/srv/readresolve.tech/magnolia/www/a-deux-pas

Back deployment dir
magnolia@vps-3229ca35:/srv/readresolve.tech/magnolia/api/a-deux-pas

HTTP access for prod https://magnolia.readresolve.tech/a-deux-pas/ serves content of /srv/readresolve.tech/magnolia/www/a-deux-pas
secured with basic auth user=webuser password => stored in jenkins credential id webuser-http-password (Secret text)

================
DEV machine localhost for front and back
Front deployment dir : ${WORKSPACE}/deploy-dev/front-a-deux-pas
Back deployment dir : ${WORKSPACE}/deploy-dev/back-a-deux-pas

HTTP URL : http://localhost

=====================

Requirements,

pipeline takes 3 params BACK_APP_VERSION FRONT_APP_VERSION TARGET_ENV ['PROD','DEV']
We deploy only the app that we have a version for
if we deploy both, start with the back first if successful deploy the front
keep it simple, straightforward readable with clear messages

make sure the target deployment dirs are empty and writable
we need to stop back app process if runing before deploying new version and then start the app

we want to load the right application.properties of the back app for the right env, see here /home/donia/Dev/a-deux-pas-fork/back-a-deux-pas/src/main/resources
we want to make sure deployment is successfull by checking the version json file of the app through http (see CI pipelines of both apps which adds these files) , to make sure the right version is live

we want to interact with env machine using regualr user no root access or sudo

lets simplify the project and use the same host for both dev and prod magnolia.readresolve.tech
we will use different dirs for bothen envs

Front PROD magnolia@vps-3229ca35:/srv/readresolve.tech/magnolia/www/a-deux-pas-prod
Front DEV magnolia@vps-3229ca35:/srv/readresolve.tech/magnolia/www/a-deux-pas-dev

Back PROD magnolia@vps-3229ca35:/srv/readresolve.tech/magnolia/api/a-deux-pas-prod
Back DEV magnolia@vps-3229ca35:/srv/readresolve.tech/magnolia/api/a-deux-pas-dev

we make sure to use differnets ports for prod and dev back apps
and each front app uses the corresponding back app url:port

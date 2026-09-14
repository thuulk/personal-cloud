# PERSONAL HOMELAB INFRASTRUCTURE 
This project began for I needed a back-up for my iphone photos and videos, then I was
introduced into immich and the advantages it gets over paying a cloud subscription at Apple or Google, so flashed an OS
on an old Raspberry-PI to install my open-source gallery. I foresaw interest on expanding the usage range of the homelab
so despite being the only app at the moment I mounted a docker container for it, as it would allow smooth installations 
in the future. Then future arrived with Navidrome to replace my Spotify suscription alongside Calibre for getting rid of 
Amazon Kindle, each app within its docker container. 

Not long ago did I discover the concept of infrastructure as code, so I am now trying thus to replicate the homelab 
infrastructure to just git clone it into another system without repeating the process by hand. 

Future implementations shall include a password manager service.


## INSTALATION GUIDE
### NAVIDROME
For spinning up the Navidrome docker container get into its root directory and create manually the data directory
and its subdirectory cache, also creating the music directory where albums and their tracks are to be stored
$ cd ~/navidrome
~/navidrome: $ mkdir data
~/navidrome: $ mkdir music
~/navidrome: $ cd data; mkdir cache; cd ..

Now with the directories created, the container is ready to be spinned 
~/navidrome: $ sudo docker compose up


### IMMICH
For spinning up the Immich container first one has to rename the file to fulfill the .env convention
$ cd ~/immich
~/immich: $ cp .env.example .env
~/immich: $ rm -rf .env.example
~/immich: $ nano .env

While using nano rename the variables DB_PASSWORD, DB_USERNAME, DB_DATABASE_NAME, and TZ according to what is 
convenient for you; you may not do it and the docker container will still spin up, nonetheless changing at least
the two first entail a good practice. There is no need to modify UPLOAD_LOCATION nor DB_DATA_LOCATION.

Now one may spin up the container
~/immich: $ sudo socker compose up


### CALIBRE
For calibre one may just spin up the container
$ cd ~/calibre; sudo docker compose up






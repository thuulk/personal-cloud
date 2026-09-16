###PERSONAL HOMELAB INFRASTRUCTURE 
This project began for I needed a back-up for my iphone photos and videos, then I was
introduced to Immich and the advantages it gets over paying a cloud subscription at Apple or Google, so I flashed an OS
on an old Raspberry Pi to install my open-source gallery. I foresaw interest on expanding the usage range of the homelab,
so I spun up a docker container for it, as it would allow smooth future installations. 
Then future arrived with Navidrome to replace my Spotify suscription alongside Calibre for getting rid of 
Amazon Kindle, each app within its docker container. 

Not long ago did I discover the concept of infrastructure as code, so I am now trying thus to replicate the homelab 
infrastructure to just git clone it into another system without repeating the process by hand. 

Future implementations shall include a password manager service.

##ARCHITECTURE

Each service runs in its own Docker Compose stack, isolated from the others. This is a deliberate
choice, not a default:

- **Failure isolation**: if Immich crashes or its Postgres instance gets corrupted, Navidrome and
  Calibre-Web keep serving unaffected. A single monolithic container running all three would turn
  any one failure into a full outage.
- **Independent updates**: each service's image can be updated on its own schedule without
  touching the others' dependencies or triggering unrelated migrations.
- **Reduced attack surface**: every container exposes only the port and volumes it needs, instead
  of one process holding access to all data at once.
- **Dedicated dependencies where required**: Immich needs Postgres as a hard dependency; coupling
  it to the same container as Navidrome or Calibre-Web would introduce a shared point of failure
  with no actual benefit.

This structure also maps directly onto the repository layout; one directory per service, one
`docker-compose.yml` per directory. Cloning the repo reproduces exactly what runs in
production, with no hidden coupling between services.

##DEPENDENCIES
###DOCKER  
To install docker follow the instructions in the official guide: https://docs.docker.com/engine/install/  
After installing one must add Docker into the group of programs your current user may access to  
```bash
$ sudo usermod -aG docker $USER
```

##INSTALLATION GUIDE
###NAVIDROME
For spinning up the Navidrome docker container get into its root directory and create manually the data directory
and its subdirectory cache, also creating the music directory where albums and their tracks are to be stored  
```bash
$ cd ~/navidrome  
~/navidrome: $ mkdir data  
~/navidrome: $ mkdir music  
~/navidrome: $ cd data; mkdir cache; cd ..
```

Now with the directories created, the container is ready to be spinned   
```bash
~/navidrome: $ docker compose up
```


###IMMICH
For spinning up the Immich container first one has to rename the file to fulfill the .env convention  
```bash
$ cd ~/immich  
~/immich: $ cp .env.example .env     
~/immich: $ nano .env
```

While using nano rename the variables DB_PASSWORD, DB_USERNAME, DB_DATABASE_NAME, and TZ according to what is
convenient for you; you may not do it and the docker container will still spin up, nonetheless changing at least
the two first entail a good practice. There is no need to modify UPLOAD_LOCATION nor DB_DATA_LOCATION.

Now one may spin up the container  
```bash
~/immich: $  docker compose up
```


###CALIBRE
For calibre one may just spin up the container  
```bash
$ cd ~/calibre; docker compose up
```


##CI/CD
The .github/workflows/validate-compose.yml workflow runs on every push or pull request that touches any of the
Docker containers (docker-compose.yml):
- **Validate** runs on GitHub-hosted runners (ubuntu-latest) and checks each service's compose file with ```bash
docker compose config
```
ensuring no syntax or identation errors in the files changed.  
- **Deploy** runs only when a push is made to main, and only after validate passes. It runs on a self-hosted runner
. A GitHub Actions agent installed directly on the Raspberry Pi, which connects outbound to GitHub rather than 
requiring any inbound port to be opened. This job copies the updated docker-compose.yml from the checked-out repo 
into each services live production directory (~/calibre, ~/immich, ~/navidrome) and runs 
```bash
docker compose up -d
```
which recreates only the containers whose configuration actually changed.  
Because **deploy** executes on the same physical machine that serves these apps, it is restricted to push events on
**main** only; never **pull_request** or **pull_request_targer**, so that a fork's pull request can never run 
arbitrary code on the host machine.

##Troubleshooting   
###Navidrome: permission denied on cache path   
**Symptom**, Container logs show repeated errors such as **Error creating cache path: permission denied**  
**Cause**, the `data/cache` directory was created with permissions or ownership that don't match
the UID/GID the Navidrome process runs as inside the container.  
**Fix**:  
```bash
cd ~/navidrome
docker compose down
sudo chown -R $USER:$USER data music
docker compose up -d
docker compose logs
```

Confirm the fix by checking the logs for `Navidrome server is ready!` with no `permission denied`
lines after it.

### Immich: ENOENT on encoded-video/.immich after moving the install directory
**Symptom**, the `immich_server` container restarts in a loop, with logs showing
**[Microservices:StorageService] Failed to read (/data/encoded-video/.immich):
Error: ENOENT: no such file or directory, open '/data/encoded-video/.immich'**  

**Cause**,on first boot, Immich writes a marker file (`.immich`) inside each of its mounted
folders (`upload`, `library`, `thumbs`, `encoded-video`, etc.) and checks for that marker on every
subsequent start, as a safeguard against accidentally pointing at the wrong volume. If
`UPLOAD_LOCATION` (or `DB_DATA_LOCATION`) in `.env` is set as an **absolute path** containing the
old directory name, renaming or moving the install folder breaks that path — Docker mounts an
empty directory in its place, the marker file is nowhere to be found, and Immich refuses to start
rather than risk operating on the wrong data.

**Fix**, check what the variables actually point to:

```bash
cat ~/immich/.env | grep -E "UPLOAD_LOCATION|DB_DATA_LOCATION"
```

If either value is an absolute path referencing the old directory name, update it to match the
current path (or switch to a relative path, e.g. `./library`, so this can't happen again on a
future rename):

```bash
nano ~/immich/.env
docker compose down
docker compose up -d
docker compose logs -f
```

Watch for `Successfully verified system mount folder checks` in the logs with no `ENOENT` errors
after it.


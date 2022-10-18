# Ubuntu Server
Highly opinionated setup catered to my needs

## On the server

- #### Install Ubuntu
  https://ubuntu.com/download/desktop
  Expected user is: `kimi450`

- #### Enable ssh server (required for ansible to work)
  https://linuxize.com/post/how-to-enable-ssh-on-ubuntu-18-04/
  ```
  sudo apt update
  sudo apt install openssh-server
  sudo systemctl enable ssh
  sudo systemctl start ssh
  sudo ufw enable
  sudo ufw allow ssh
  ```

## On the client

- #### Enable passwordless ssh access from remote machine to server (required for ansible to work)
  https://www.linuxbabe.com/linux-server/setup-passwordless-ssh-login
  ```
  ssh-keygen -t rsa -b 4096
  file ~/.ssh/id_rsa
  ssh-copy-id -p <ssh-port> <remote-user>@<server-ip>
  ```

- #### Update the hosts.yaml file to update the IP of the server

- #### Install ansible
  ```
  sudo apt update
  sudo apt install software-properties-common
  sudo add-apt-repository --yes --update ppa:ansible/ansible
  sudo apt install ansible
  ```

- #### Get the ``Zone`` name from Cloudflare where your DNS entries are present and an API token that can edit DNS entries and read Zone information [OPTIONAL]
  - https://dash.cloudflare.com/profile/api-tokens
  - If not needed, remove the line below line from `setup.yaml`
  `- import_playbook: install-and-configure-cloudflare-dns-updater-service.yaml`

- #### Expose required ports on your router
  - Expose (port forward on your router) ports for the services you wish to have available externally based on the list [here](#exposed-services).

- #### Run the ansible runner script
  - `./run.sh`

- #### After the installation
  - ##### Setup Grafana
    - Change the default login details 
    - Add the recommended dashboards
      - [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
      - [Pods (Aggregated view)](https://grafana.com/grafana/dashboards/8860)
      - [Monitor Pod CPU and Memory usage](https://grafana.com/grafana/dashboards/15055)
      - [Node Exporter for Prometheus Dashboard EN v20201010](https://grafana.com/grafana/dashboards/11074)

  - ##### Setup Jellyfin
    - Initial setup is just following on-screen instructions.
      - If asked to select server, delete it and refresh the page.
    - Point Jellyfin to use the directories mentioned in the playbooks for shows, movies and music.
      - By default, on the Jellyfin pod, the directories it will be:
        ```
        /media/data/shows
        /media/data/movies
        /media/data/music
        ```
    - Add any other config required.
      - Recommend setting up the Open Subtitles plugin which requires creating an account on [their website](https://www.opensubtitles.org/en/?).
      - For Hardware acceleration go to ``Admin > Dashboard > Playback``
          - Enable ``Hardware acceleration``
          - Select ``Video Acceleration API (VAAPI)`` which is setup already to use the **integrated Intel GPU**. Not tested with anything else (like a dedicated AMD/Nvidea GPU)
            - You should see CPU usage drop and GPU usage go up, disable it if you dont or troubleshoot.
            - You can use the ``intel-gpu-tools`` package to monitor (notice GPU usage when hardware encoding is enabled, and no GPU usage when it is disabled) at least the intel GPU by running the command below on the host:
              ``sudo intel_gpu_top``
          - Select the formats for which hardware acceleration should be enabled
            - Recommend not selecting ```HEVC 10bit``` because for some reason that breaks it
          - Defaults to CPU/software encoding if hardware acceleration does not work for a file, I think.
          - More infomarmation on their [Jellyfin's page for Hardware Acceleration](https://jellyfin.org/docs/general/administration/hardware-acceleration.html)
  - ##### Setup qBittorrent
    - Default login credentials are admin/adminadmin
    - Change the default login details
      - Go to ``Tools > Options > Web UI > Authentication``
    - Set default download location to one the mentioned directories (or make sure to put it in the right directory when downloading for ease)
      - Recommend using ``/media/data/downloads``
    - Set seeding limits
      - Recommend seeding limits for when seeding ratio hits "0". It is under ``Tools > Options > BitTorrent > Seeding Limits``
    - Set torrent download/upload limits
      - Recommended to keep 6 active torrents/downloads and 0 uploads. It is under ``Tools > Options > BitTorrent > Torrent Queueing``

  - ##### Setup Calibre
    - Do base setup
      - Set folder to be ``/media/data/books`` and select ``Yes`` for it to rebuild the library if asked.
    - Go to ``Preferences > Sharing over the net`` (click on the 3 dots on the top right)
      - Check the box for ``Require username and password to access the Content server``
      - Check the box for ``Run the server automatically when calibre starts``
      - Click on ``Start server``
      - Go to the ``User accounts tab`` and create a user
        - Make a note of the credentials for use in ``Readarr`` setup
      - Restart the app/pod

  - ##### Setup Calibre Web
    - Set folder to be ``/media/data/books``
    - Default login is ``admin/admin123``
    - To enable web reading, click on ``Admin`` on the top right
      - Click on the user, default is ``admin``
      - Click on ``Allow ebook viewer``
      - Change password to something more secure

  - ##### Setup Radarr/Sonarr/Readarr/Lidarr
    - Service function

      | Service | Purpose  |
      |---------|----------|
      | Readarr | Books    |
      | Sonarr  | TV Shows |
      | Radarr  | Movies   |
      | Lidarr  | Music    |

    - Go to ``Settings`` and click on ``Show Advanced``
    - Enable authentication
      - Go to ``Settings > General``
      - Set Authentication to `Forms (Login Page)`
      - Set username and password for access
    - Add torrent client
      - Go to ``Settings > Download Clients > Add > qBittorent > Custom``
      - Add the host: ``qbittorrent``
      - Add the port: ``8080``
      - Add the username: ``<qBittorrent_username>``
      - Add the password: ``<qBittorrent_password>``
      - Uncheck the ``Remove Completed`` option.
        - When enabled, this seems to delete the downloaded files sometimes. Not sure why.
    - Set the root directories to be the following
      - Go to ``Settings > Media Management``

        | Service | Root Directory          |
        |---------|-------------------------|
        | Readarr | ``/media/data/books/``  |
        | Sonarr  | ``/media/data/shows/``  |
        | Radarr  | ``/media/data/movies/`` |
        | Lidarr  | ``/media/data/music/``  |
      - Enable renaming

    - Readarr specific config
      - Go to ``Settings > Media Management``
        - Add root folder
          - Set the path to be ``/media/data/books/``
          - Enable ``Use Calibre`` options the the following defaults
            - Calibre host: ``calibre-webserver``
            - Calibre port: ``8081``
            - Calibre Username: ``<calibre_username>``
            - Calibre Password: ``<calibre_password>``
        - Enabled ``Rename Books`` and use the defaults

  - ##### Setup Ombi
    - One stop shop for Sonarr/Radarr/Lidarr requests
    - Get the API keys for Jellyfin, Sonarr and Radarr
      - Jellyfin
        - Go to ``Admin > Dashboard > API Keys``
        - Generate a new API key with an appropriate name
      - Sonarr/Radarr/Lidarr
        - Use the API tokens from the respective services, found under ``Settings > General > Security > API Key`` 
    - Use the API key for Jellyfin at the first time setup, dont use SSL.
      | Service Name | Port |
      |--------------|------|
      | jellyfin     | 8096 |

    - Set credentials for login
    - Go to ``Settings``
      - Use the correct API keys, hostnames and ports for the services
          | Service Name | Port |
          |--------------|------|
          | sonarr       | 8989 |
          | radarr       | 7878 |
          | lidarr       | 8686 |
      - Click on the ``Load Profiles`` and ``Load Root Folders`` buttons and use the appropriate defaults as used in the services seen [here](#setup-radarrsonarrreadarrlidarr).
      - Setup ``Movies`` using ``Radarr``
      - Setup  ``TV`` using ``Sonarr``
        - Enable the ``Enable season folders`` option
        - Enable the ``V3`` option
      - Setup  ``Music`` using ``Lidarr``
      - Dont forget to click on ``Enable`` for each of those setups as well
    - Go to ``Users``
      - Setup additional users
      - Give the following roles to *trusted* users for convinience
        ```
        AutoApproveMovie
        AutoApproveMusic
        AutoApproveTv
        ```

  - ##### Setup Prowlarr
    - Enable authentication
      - Go to ``Settings > General``
      - Set Authentication to `Forms (Login Page)`
      - Set username and password for access
    - Follow the [official Quick Start Guide](https://wiki.servarr.com/prowlarr/quick-start-guide)
      - Add all the indexers you wish to use, some good ones listed below. Find more indexers on [Prowlarr's Supported Indexers page](https://wiki.servarr.com/prowlarr/supported-indexers).
        - Standard
          ```
          1337x
          Rarbg
          showRSS
          LimeTorrents
          ```
        - Anime
          ```
          Nyaa.si
          ```
    - Add Sonarr, Radarr and Readarr to the ``Settings > Apps > Application`` section using the correct API token and kubernetes service names
      - By default the services will be ``http://sonarr:8989``, ``http://radarr:7878`` and ``http://readarr:8787``

  - ##### Setup Bazarr
    - Enable authentication
      - Go to ``Settings > General``
      - Under ``Security`` select ``Form`` as the form of ``Authentication``
      - Set username and password for access
    - Follow this page
      - ``https://wiki.bazarr.media/Getting-Started/Setup-Guide/``
      - Go to ``Settings > Radarr`` and ``Settings > Sonarr``
        - Fill out the details and save
          - Use the API tokens from the respective services, found under ``Settings > General > Security > API Key``
          - Use the kubernetes service name and port

            | Service Name | Port |
            |--------------|------|
            | radarr       | 7878 |
            | sonarr       | 8989 |

        - Fill out the path mappings if the directories in which data is stored is different for both services (same by default)
      - Go to ``Settings > Languages``
        - Add a language profile and set defaults for movies and series'
      - Go to ``Settings > Provider`` and add providers for subtitles
        - Decent options are:
          - Opensubtitles.com
          - TVSubtitles
          - YIFY Subtitles
          - Supersubtitles
      - Go to ``Settings > Subtitles`` and make changes if needed
      - Manually add the language profile to all the scanned media after first installation
    - NOTE:
      - If it doesnt work, manually reinstall this service a few times. It just works, not sure whhy

  - ##### Setup Minikube for remote access
    - Move the copied `minikube_client.crt` and `minikube_client.crt` file to appropriate locations
    - Edit the `minikube_config` file copied over to:
      - Reflect the new locations of the `minikube_client.crt` and `minikube_client.crt` files
      - Change the server address to something public facing if needed and change protocol if needed
    - Edit your local `~/.kube/config` and incorporate the information from the `minikube_config` into it

  - ##### Use Squid
    - Use the username and password passed in during installation to use this as a proxy server
    - The address would be ``<PUBLIC_IP>:3128`` or ``<DOMAIN_NAME>:3128`` or ``<LAN_IP>:3128``

  - ##### Exposed Services
    - You can port forward the following ports on your router to gain external.
    - You need to create DNS entries to access the Ingress services. The following entries are recommended:
      - ``*.<DOMAIN_NAME>``
      - ``<DOMAIN_NAME>``

      | Service     | Default access | Where                                       | Port to be forwarded from server |
      |-------------|----------------|---------------------------------------------|----------------------------------|
      | ssh         | ssh            | ``<LAN_IP>`` or ``<DOMAIN_NAME>``           |  22                              |
      | minikube    | api-access     | ``<LAN_IP>`` or ``<DOMAIN_NAME>``           |  3001                            |
      | squid       | proxy          | ``<LAN_IP>:3128`` or ``<DOMAIN_NAME>:3128`` | 3128                             |
      | grafana     | Ingress        | ``grafana.<DOMAIN_NAME>``                   |  80                              |
      | jellyfin    | Ingress        | ``jellyin.<DOMAIN_NAME>``                   |  80                              |
      | ombi        | Ingress        | ``ombi.<DOMAIN_NAME>``                      |  80                              |
      | prowlarr    | Ingress        | ``prowlarr.<DOMAIN_NAME>``                  |  80                              |
      | bazarr      | Ingress        | ``bazarr.<DOMAIN_NAME>``                    |  80                              |
      | radarr      | Ingress        | ``radarr.<DOMAIN_NAME>``                    |  80                              |
      | sonarr      | Ingress        | ``sonarr.<DOMAIN_NAME>``                    |  80                              |
      | readarr     | Ingress        | ``readarr.<DOMAIN_NAME>``                   |  80                              |
      | lidarr      | Ingress        | ``lidarr.<DOMAIN_NAME>``                    |  80                              |
      | librespeed  | Ingress        | ``librespeed.<DOMAIN_NAME>``                |  80                              |
      | calibre-web | Ingress        | ``calibre-web.<DOMAIN_NAME>``               |  80                              |
      | calibre     | LAN            | ``<LAN_IP>:3002``                           | 3002                             |

      NOTE: Security is an unkown when exposing a service to the internet.


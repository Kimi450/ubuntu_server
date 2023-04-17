# Ubuntu Server
Highly opinionated server setup to cater to my needs

## Server Setup

### Cloud
Probably get a VM from [Oracle Always Free Tier](https://www.oracle.com/ie/cloud/free/) stuff.

### On-Prem
Use your own server

- #### Install Ubuntu
  https://ubuntu.com/download/desktop

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

## Client Setup

- #### Enable passwordless ssh access from remote machine to server (required for ansible to work)
  ```
  ssh-keygen -t ed25519 -C "primary-key"
  file ~/.ssh/id_ed25519.pub
  ssh-copy-id -p <ssh-port> <remote-user>@<server-ip>
  ```

  If youre using a machine that only allows for publickey auth, then you can upload your key that you just generated with the following command

  ```
  ssh-copy-id -i ~/.ssh/id_ed25519.pub -o 'IdentityFile ~/.ssh/<your-existing-private-key-for-access>.key' -p <ssh-port> <remote-user>@<server-ip>
  ```

- #### Install ansible
  ```
  sudo apt update
  sudo apt install software-properties-common
  sudo add-apt-repository --yes --update ppa:ansible/ansible
  sudo apt install ansible
  ```

- #### Update the `hosts.yaml` file to fill out the template

- #### Update the `group_vars/all` file to fill out the required information there
  At the very least, search for the items with tags `# FILL OUT`

  - ##### Get CloudFlare information [OPTIONAL]
    - This is if you use CloudFlare for your domain name and want the public IP updated automatically every set amount of time incase it is changed on the server.
    - **NOTE:** Make sure the services installed in this dont overlap with any existing implied DNS entries you may have (keep this in mind if you see any weird behaviour, im not an expert here and im too tired to think about this at the moment)
    - Register your domain name on CloudFlare
    - Go to the main CloudFlare page
    - Then `Websites`
    - Then select the relevant `Zone` (basically the website you used in the `group_vals/all` file)
    - Go to the `Overview` page
      - On the right side you can see the `Zone ID`
      - Put this in the `group_vars/all` file
      - Here you can also see the link to the API token page
    - Go to the `DNS` page 
      - Put in the following records (**REQUIRED**)

        | Type | Name                   | Content            | Proxy Status | TTL    |
        |------|------------------------|--------------------|--------------|--------|
        | `A`  | `<YOUR_DOMAIN_NAME>`   | `<YOUR_PUBLIC_IP>` | `DNS only`   | `Auto` |
        | `A`  | `*.<YOUR_DOMAIN_NAME>` | `<YOUR_PUBLIC_IP>` | `DNS only`   | `Auto` |
  
      - Now you can also use `<YOUR_DOMAIN_NAME>` in the `group_vars/all` file instead of the server's IP address
    - Create a Custom API token from the [api-tokens](https://dash.cloudflare.com/profile/api-tokens) page with the following permissions and include the specific `Zone` (or website) from `Zone Resources` section
      - To edit DNS entries
        - `Zone:DNS:Edit`
      - To read `Zone` information
        - `Zone:Zone:Read`
      - Put this token/key in the `group_vars/all` file
    - If not needed, remove the line below line from `setup.yaml`
    `- import_playbook: install-and-configure-cloudflare-dns-updater-service.yaml`

- #### Expose required ports on your router
  - Expose (port forward on your router) ports for the services you wish to have available externally based on the list [here](#exposed-services).

- #### Run the ansible runner script
  - `./run.sh`
  - You can add `-vvvv` to get more verbose output

- #### After the installation
  - ##### Setup Grafana
    - Add the recommended dashboards
      - [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
      - [Pods (Aggregated view)](https://grafana.com/grafana/dashboards/8860)
      - [Monitor Pod CPU and Memory usage](https://grafana.com/grafana/dashboards/15055)
      - [Node Exporter for Prometheus Dashboard EN v20201010](https://grafana.com/grafana/dashboards/11074)
      - [Loki Kubernetes Logs](https://grafana.com/grafana/dashboards/15141)
    - Would recommend adding a panel with the following query as it is useful to monitor pods as well
      - For sum (Probably better)
        ```
        sum (irate(container_cpu_usage_seconds_total[2m])) by (pod)
        ```
      - For average
        ```
        avg(irate(container_cpu_usage_seconds_total[2m])) by (pod,container)
        ```
    - You can find information on how to use [Loki](https://grafana.com/oss/loki/) in Grafana [here](https://grafana.com/docs/loki/latest/operations/grafana/)

  - ##### Setup Jellyfin
    - Initial setup is just following on-screen instructions.
      - If asked to select server, delete it and refresh the page.
    - Point Jellyfin to use the directories mentioned in the playbooks for shows, movies, music and books.
      - By default, on the Jellyfin pod, the directories it will be:
        ```
        /media/data/shows
        /media/data/movies
        /media/data/music
        /media/data/books
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
    - Go to ``Preferences > Sharing over the net``
      - Check the box for ``Require username and password to access the Content server``
      - Check the box for ``Run the server automatically when calibre starts``
      - Click on ``Start server``
      - Go to the ``User accounts tab`` and create a user
        - Make a note of the credentials for use in ``Readarr`` setup
      - Restart the app/pod
        - You can do so by also pressing `CTRL + R` on the main screen

  - ##### Setup Calibre Web
    - Default login is ``admin/admin123``
    - Set folder to be ``/media/data/books``
    - To enable web reading, click on ``Admin`` (case sensitive) on the top right
      - Click on the user, default is ``admin``
      - Enable ``Allow ebook viewer``
      - Change password to something more secure
      - Save settings

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
      - Add the port: ``10095``
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
        - Add root folder (you cannot edit an existing one)
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
          LimeTorrents
          The Pirate Bay
          EZTV
          ```
        - Anime
          ```
          Nyaa.si
          ```
    - Add Sonarr, Radarr, Lidarr and Readarr to the ``Settings > Apps > Application`` section using the correct API token and kubernetes service names
      - By default the services will be ``http://sonarr:8989``, ``http://radarr:7878``, ``http://lidarr:8686`` and ``http://readarr:8787``
      - Select extra `Sync Catagories` for each application if required
        - If you dont know what to do, add all of them for every app (comes at the cost of slower searches)

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
    - Use the kubeconfig file copied over to the current working directory by exporting it
      - `export KUBECONFIG=<KUBECONFIG_LOCATION>`
    - Optionally, edit your local `~/.kube/config` and incorporate the information from the copied over kubeconfig into it
    - **NOTE:**
      - The port on which kube-apiserver is forwarded to, 3001 by default, should not be exposed to the internet (i.e., should be LAN access only) because anyone will be able to access it.
      - The way it is set up at the moment, the certs dont really do anything. The apiserver itself is directly accessible without any authentication.
        - See [issue #12](https://github.com/Kimi450/ubuntu_server/issues/12)).
      - By default, `ansible_host` from the `hosts.yaml` file is used as the IP in the kubeconfig file. It is **strongly recommended** that you change that to the LAN IP of the server (to not have to port forward this on your router to access it)

  - ##### Use Squid
    - Use the username and password from the `group_vars/all` file to use this as a proxy server
    - The address would be `<PUBLIC_IP>:<GROUP_VARS_PORT>` or `<DOMAIN_NAME>:<GROUP_VARS_PORT>` or `<LAN_IP>:<GROUP_VARS_PORT>`

  - ##### Use Sambashare
    - For external access:
      - The following info was retrieved by running `sudo ufw status verbose | grep -i samba` on the server which lists what ports were exposed as part of `sudo ufw allow samba`
      - Expose the following ports for TCP
        ```
        139
        445
        ```
      - Expose the following ports for UDP
        ```
        137
        138
        ```
    - To authenticate
      - Thee username will be the `<ANSIBLE_USER>` you used in the `hosts.yaml` file
      - The password will be in the `group_vars/all` file (`smb.password` section).
    - In Windows, connect to it using `\\<LAN_IP>\<SHARE_NAME_FROM_GROUP_VARS_ALL>`
    - More information [here](https://ubuntu.com/tutorials/install-and-configure-samba#4-setting-up-user-accounts-and-connecting-to-share)

  - ##### Exposed Services
    - You can port forward the following ports on your router to gain external.
    - You need to create DNS entries to access the Ingress services. The following entries are recommended:
      - `*.<DOMAIN_NAME>`
      - `<DOMAIN_NAME>`

      | Service     | Default access | Where                                                             | Port to be forwarded from server |
      |-------------|----------------|-------------------------------------------------------------------|----------------------------------|
      | ssh         | ssh            | `<LAN_IP>` or `<DOMAIN_NAME>`                                     |                               22 |
      | squid       | proxy          | `<LAN_IP>:<GROUP_VARS_PORT>` or `<DOMAIN_NAME>:<GROUP_VARS_PORT>` |                             3128 |
      | samba       | proxy          | `\\<LAN_IP>\<SHARE_NAME>` or `\\<DOMAIN_NAME>\<SHARE_NAME>`       |   TCP: `139,445`, UDP: `137,138` |
      | grafana     | Ingress        | `grafana.<DOMAIN_NAME>`                                           |                               80 |
      | jellyfin    | Ingress        | `jellyin.<DOMAIN_NAME>`                                           |                               80 |
      | ombi        | Ingress        | `ombi.<DOMAIN_NAME>`                                              |                               80 |
      | prowlarr    | Ingress        | `prowlarr.<DOMAIN_NAME>`                                          |                               80 |
      | bazarr      | Ingress        | `bazarr.<DOMAIN_NAME>`                                            |                               80 |
      | radarr      | Ingress        | `radarr.<DOMAIN_NAME>`                                            |                               80 |
      | sonarr      | Ingress        | `sonarr.<DOMAIN_NAME>`                                            |                               80 |
      | readarr     | Ingress        | `readarr.<DOMAIN_NAME>`                                           |                               80 |
      | lidarr      | Ingress        | `lidarr.<DOMAIN_NAME>`                                            |                               80 |
      | librespeed  | Ingress        | `librespeed.<DOMAIN_NAME>`                                        |                               80 |
      | calibre-web | Ingress        | `calibre-web.<DOMAIN_NAME>`                                       |                               80 |
      | minikube    | LAN api-access | `<LAN_IP>:3001`                                                   |                             3001 |
      | calibre     | LAN            | `<LAN_IP>:3002`                                                   |                             3002 |

      NOTE: Security is an unkown when exposing a service to the internet.


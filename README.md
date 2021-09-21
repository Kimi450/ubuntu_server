# Setup
### On the server

- Install Ubuntu
  - https://ubuntu.com/download/desktop
  - Expected user is: `kimi450`

- Enable ssh server (required for ansible to work)
  - https://linuxize.com/post/how-to-enable-ssh-on-ubuntu-18-04/
  -  ```
     sudo apt update
     sudo apt install openssh-server
     sudo systemctl enable ssh
     sudo systemctl start ssh
     sudo ufw enable
     sudo ufw allow ssh
     ```

### On the client

- Enable passwordless ssh access from remote machine to server (required for ansible to work)
    https://www.linuxbabe.com/linux-server/setup-passwordless-ssh-login
    ```
    ssh-keygen -t rsa -b 4096
    file ~/.ssh/id_rsa
    ssh-copy-id <remote-user>@<server-ip>
    ```

- Update the hosts.yaml file to update the IP of the server

- Install ansible
  - ```
    sudo apt update
    sudo apt install software-properties-common
    sudo add-apt-repository --yes --update ppa:ansible/ansible
    sudo apt install ansible
    ```

- Run the ansible runner script
  - `./run.sh`

- After the installation

  - Setup Grafana 
    - Change the default login details 
    - Add the recommended charts

  - Setup Jellyfin
    - Setting up a login 
    - Point Jellyfin to use the directories mentioned in the playbooks for shows and movies.
      - By default, on the Jellyfin pod, the directories it will be:
        ```
        /media/data/shows
        /media/data/movies
        ```
    - Add any other config required.

  - Setup qBittorrent
    - Change the default login details 
    - Set default download location to one the mentioned directories (or make sure to put it in the right directory when downloading for ease)
      - Recommend using ``/media/data/``
    - Set seeding limits
      - Recommend seeding limits for when seeding ratio hits "0". It is under ``Options > BitTorrent``
    - Set torrent download/upload limits
     - Recommended to keep 6 active torrents/downloads and 0 uploads. It is under ``Options > BitTorrent``
  - Setup Jackett
    - Add all the indexers you wish to use.
    - Make a note of the API key

  - Setup Radarr/Sonarr
    - Go to ``Settings`` and click on ``Show Advanced``
    - Go to ``Settings > Indexers > Add > Torznab > Custom``
      - Add the URL: ``http://jackett:9117``
      - Add the API Path: ``/api/v2.0/indexers/all/results/torznab``
      - NOTE: Or you may need to concatinate both of them if you dont click on ``Show Advanced``
        - ``http://jackett:9117/api/v2.0/indexers/all/results/torznab``
      - Set Minimum Seeders to an appropriate value
        - It is ``1`` by default, you can change it to probably ``5`` but this will have implications, i.e., sometimes not being able to find anything to download.
    - Go to ``Settings > Download Clients > Add > qBittorent > Custom``
      - Add the host: ``qbittorrent``
      - Add the port: ``8080``
      - Add the username: ``<qBittorrent_username>``
      - Add the password: ``<qBittorrent_password>``
      - Uncheck the ``Remove Completed`` option.
        - When enabled, this seems to delete the downloaded files sometimes. Not sure why.
    - Set the base download location to be one of the following
      - Radarr:``/media/data/movies/``
      - Sonarr:``/media/data/shows/``

# Exposed services
You can port forward the following ports on your router to gain external access as well.

NOTE: Security is an unkown when exposing a service to the internet.

### Kubernetes API server
The kubernetes API server is accessible on `<SERVER_IP>:3001` through the client on the same LAN.

### Grafana
Grafana can be accessed on `<SERVER_IP>:3002` through the client machine on the same LAN.

### Jellyfin
Jellyfin can be accessed on `<SERVER_IP>:3003/web/index.html` through the client machine on the same LAN.

### qBittorrent
qBittorrent can be accessed on `<SERVER_IP>:3004` through the client machine on the same LAN.

### Jackett
Jackett can be accessed on `<SERVER_IP>:3005` through the client machine on the same LAN.

### Radarr
Radarr can be accessed on `<SERVER_IP>:3006` through the client machine on the same LAN.

### Sonarr
Sonarr can be accessed on `<SERVER_IP>:3007` through the client machine on the same LAN.


# Appendix

### Good dashboards
- [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
- [Node Exporter for Prometheus Dashboard EN v20201010](https://grafana.com/grafana/dashboards/11074)

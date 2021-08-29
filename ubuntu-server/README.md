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
- Disable sleep/suspend
  -  ```sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target```

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

# Exposed services
### Grafana
Grafana can be accessed on `<SERVER_IP>:3000` through the client on the same LAN.
#### Good dashboards
- [Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
### Kubernetes API server
The kubernetes API server is accessible on `<SERVER_IP>:6969` through the client on the same LAN/

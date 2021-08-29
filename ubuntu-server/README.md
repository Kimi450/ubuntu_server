# How to set it up?
## On the server
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

## On the client
- Enable passwordless ssh access from remote machine to server (required for ansible to work)
    https://www.linuxbabe.com/linux-server/setup-passwordless-ssh-login
    ```
    ssh-keygen -t rsa -b 4096
    file ~/.ssh/id_rsa
    ssh-copy-id <remote-user>@<server-ip>
    ```
- Update the hosts.yaml file to update the IP of the server
- Run the ansible runner script
  - `./run.sh`
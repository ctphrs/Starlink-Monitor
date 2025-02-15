# Internet Pi

[![CI](https://github.com/geerlingguy/internet-pi/workflows/CI/badge.svg?event=push)](https://github.com/geerlingguy/internet-pi/actions?query=workflow%3ACI)

**A Raspberry Pi Configuration for Internet connectivity**

This is a fork of geerlinguy's internet pi with a focus on testing the network performance of temporary test networks rather than a home network with users. I have added a smokeping docker image to run and plan to add iPerf support as well, i do not plan on using Pi-hole and might remove it but see no issue with just leaving it disabled in the config for now.

## Features

**Internet Monitoring**: Installs Prometheus and Grafana, along with a few Docker containers to monitor your Internet connection with Speedtest.net speedtests and HTTP tests so you can see uptime, ping stats, and speedtest results over time.

![Internet Monitoring Dashboard in Grafana](/images/internet-monitoring.png)

**Smoke Ping**: Runs [Smokeping Docker Image](https://oss.oetiker.ch/smokeping) to ping configured targets and track detailed response time data.

![Smoke Ping on the Internet Pi](/images/smokeping.png)

Other features:

  - **Shelly Plug Monitoring**: Installs a [`shelly-plug-prometheus` exporter](https://github.com/geerlingguy/shelly-plug-prometheus) and a Grafana dashboard, which tracks and displays power usage on a Shelly Plug running on the local network. (Disabled by default. Enable and configure using the `shelly_plug_*` vars in `config.yml`.)
  - **AirGradient Monitoring**: Configures [`airgradient-prometheus`](https://github.com/geerlingguy/airgradient-prometheus) and a Grafana dashboard, which tracks and displays air quality over time via one or more AirGradient DIY monitors. (Disabled by default. Enable and configure using the `airgradient_enable` var in `config.yml`. See example configuration for ability to monitor multiple AirGradient DIY stations.)
  - **Starlink Monitoring**: Installs a [`starlink` prometheus exporter](https://github.com/danopstech/starlink_exporter) and a Grafana dashboard, which tracks and displays Starlink statistics. (Disabled by default. Enable and configure using the `starlink_enable` var in `config.yml`.)

## Recommended Pi and OS

You should use a Raspberry Pi 4 model B or better. The Pi 4 and later generations of Pi include a full gigabit network interface and enough I/O to reliably measure fast Internet connections.

Older Pis work, but have many limitations, like a slower CPU and sometimes very-slow NICs that limit the speed test capability to 100 Mbps or 300 Mbps on the Pi 3 model B+.

Other computers and VMs may run this configuration as well, but it is only regularly tested on a Raspberry Pi.

The configuration is tested against Raspberry Pi OS, both 64-bit and 32-bit, and runs great on that or a generic Debian installation.

It should also work with Ubuntu for Pi, or Arch Linux, but has not been tested on other operating systems.

## Setup

  1. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html). The easiest way (especially on Pi or a Debian system) is via Pip:
     1. (If on Pi/Debian): `sudo apt-get install -y python3-pip`
     2. (on Pi/Debian and Everywhere): `pip3 install ansible`
     3. if that doesn't work just use apt-get to install ansible `sudo apt-get install ansible`
  2. Install Git `sudo apt-get install git` Clone this repository: `git clone https://github.com/ctphrs/Starlink-Monitor.git`, then enter the repository directory: `cd Starlink-Monitor`.
  3. Install requirements: `ansible-galaxy collection install -r requirements.yml` (if you see `ansible-galaxy: command not found`, restart your SSH session or reboot the Pi and try again)
  4. Make copies of the following files and customize them to your liking:
     - `example.inventory.ini` to `inventory.ini` (replace IP address with your Pi's IP, or comment that line and uncomment the `connection=local` line if you're running it on the Pi you're setting up).
     - `example.config.yml` to `config.yml`
  5. Run the playbook: `ansible-playbook main.yml`
  6. Install the emulator to run the weather exporter on a pi `docker run --privileged --rm tonistiigi/binfmt --install amd64`

> **If running locally on the Pi**: You may encounter an error like "Error while fetching server API version". If you do, please either reboot or log out and log back in, then run the playbook again.

## Usage

### Grafana

Visit the Pi's IP address with port 3030 (e.g. http://192.168.1.10:3030/), and log in with username `admin` and the password `monitoring_grafana_admin_password` you configured in your `config.yml`.

To find the dashboard, navigate to Dashboards, click Browse, then go to the Internet connection dashboard. If you star this dashboard, it will appear on the Grafana home page.

> Note: The `monitoring_grafana_admin_password` is only used the first time Grafana starts up; if you need to change it later, do it via Grafana's admin UI.


### SmokePing

Visit the Pi's IP address and with the port 80 (e.g. http://192.168.1.10:80/), This will provide access to the SmokePing webpage.

The Smokeping docker image is https://hub.docker.com/r/linuxserver/smokeping. So reference this to customize the service.

Once the playbook is run for the first time a targets file will be created at /home/pi/internet-monitoring/smoke_ping/config/Targets

Navigate to is and edit it to your liking following the format shown below

```
*** Targets ***
probe = FPing
menu = Top
title = Network Latency Grapher
remark = Welcome to the SmokePing website. Here you will learn about latency in the network.

+ InternetSites
menu = Internet Sites
title = Internet Sites

++ Youtube
menu = YouTube
title = YouTube
host = youtube.com
```
In order for the change to take effect you might need to delete the database file and restart the playbook

### Prometheus

A number of default Prometheus job configurations are included out of the box, but if you would like to add more to the `prometheus.yml` file, you can add a block of text that will be added to the end of the `scrape_configs` using the `prometheus_extra_scrape_configs` variable, for example:

```yaml
prometheus_extra_scrape_configs: |
  - job_name: 'customjob'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.1.1:9100']
```

You can also add more targets to monitor via the node exporter dashboard, say if you have a number of servers or other Pis you want to monitor on this instance. Just add them to the list, after the `nodeexp:9100` entry for the main Pi:

```yaml
prometheus_node_exporter_targets:
  - 'nodeexp:9100'
  # Add more targets here
  - 'another-server.local:9100'
```

### Running on Startup

To make an Ansible playbook run on startup on a Raspberry Pi, you can use a combination of systemd and Ansible. Here's a step-by-step guide to accomplish this:

1. **Write Your Ansible Playbook**: Create an Ansible playbook that contains the tasks you want to run on startup. Save this playbook on your Raspberry Pi.

2. **Create a Systemd Service Unit**:

    a. Create a new systemd service unit file. You can use any text editor you prefer, like `nano` or `vim`. For example, let's create a service unit named `my_ansible_playbook.service`:

    ```shell
    sudo nano /etc/systemd/system/my_ansible_playbook.service
    ```

    b. Add the following content to the `my_ansible_playbook.service` file, replacing `<PATH_TO_YOUR_PLAYBOOK>` with the actual path to your Ansible playbook:

    ```ini
    [Unit]
    Description=Run My Ansible Playbook at Startup
    After=network.target

    [Service]
    ExecStart=/usr/bin/ansible-playbook <PATH_TO_YOUR_PLAYBOOK>
    WorkingDirectory=/path/to/playbook/directory
    User=<YOUR_USER>
    Group=<YOUR_GROUP>
    Restart=always

    [Install]
    WantedBy=multi-user.target
    ```

    - `Description`: A description of your service.
    - `ExecStart`: The command to execute your Ansible playbook.
    - `WorkingDirectory`: The directory where your playbook is located.
    - `User` and `Group`: Specify the user and group that should run the playbook (replace `<YOUR_USER>` and `<YOUR_GROUP>` with appropriate values).
    - `Restart`: Configures the service to restart always.

3. **Reload systemd**: After creating the service unit file, reload the systemd manager configuration:

    ```shell
    sudo systemctl daemon-reload
    ```

4. **Enable the Service**: Enable your service to run at startup:

    ```shell
    sudo systemctl enable my_ansible_playbook.service
    ```

5. **Start the Service**: Start the service manually to test it:

    ```shell
    sudo systemctl start my_ansible_playbook.service
    ```

6. **Reboot the Raspberry Pi**: To ensure that your Ansible playbook runs at startup, reboot your Raspberry Pi and check if the service starts automatically:

    ```shell
    sudo reboot
    ```

Your Ansible playbook should now run automatically at startup on your Raspberry Pi. Make sure to replace `<PATH_TO_YOUR_PLAYBOOK>`, `<YOUR_USER>`, and `<YOUR_GROUP>` with your specific playbook path and user/group details.

### Problems losing IPv4 address

If you have problems where the Pi loses the IPv4 address but still is running and reachable via IPv6 this is an issue with docker on a Raspian and solutions are discussed in [this](https://raspberrypi.stackexchange.com/questions/136320/raspberry-pi-loses-ipv4-address-randomly-but-keeps-ipv6-address) article.
The short of it though is this.

dhcpcd can be flooded when renewing IP addresses if too many interfaces are present. This is the case when Docker is installed and many containers/networks/services are running. In addition, docker takes care of IP addresses and routing on its virtual network, so DHCPCD doesn't need to handle them.

The solution is to configure dhcpcd to ignore all interfaces whose names start with `veth` (Docker virtual interfaces).

Edit `/etc/dhcpcd.conf` and append the following line to the end
```
denyinterfaces veth*
```
Then restart the service:
```bash
sudo systemctl restart dhcpcd.service
```
## Updating

### Configurations and internet-monitoring images

Upgrades for the other configurations are similar (go into the directory, and run the same `docker-compose` commands. Make sure to `cd` into the `config_dir` that you use in your `config.yml` file. 

Alternatively, you may update the initial `config.yml` in the the repo folder and re-run the main playbook: `ansible-playbook main.yml`. At some point in the future, a dedicated upgrade playbook may be added, but for now, upgrades may be performed manually as shown above.

## Backups

A guide for backing up the configurations and historical data will be posted here as part of [Issue #194: Create Backup guide](https://github.com/geerlingguy/internet-pi/issues/194).

## Uninstall

To remove `internet-pi` from your system, run the following commands (assuming the default install location of `~`, your home directory):

```bash
# Enter the internet-monitoring directory.
cd ~/internet-monitoring

# Shut down internet-monitoring containers and delete data volumes.
docker-compose down -v

# Shutdown pi-hole containers and delete data volumes.
docker-compose down -v

# Delete all the unused container images, volumes, etc. from the system.
docker system prune -f
```

Do the same thing for any of the other optional directories added by this project (e.g. `shelly-plug-prometheus`, `starlink-exporter`, etc.).

You can then delete the `internet-monitoring`, `pi-hole`, etc. folders and everything will be gone from your system.

## License

MIT

## Author

This project was created in 2021 by [Jeff Geerling](https://www.jeffgeerling.com/).
Forked and Modified in 2022 by Danny Williams.

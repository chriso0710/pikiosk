# Automate Chromium in kiosk mode on Raspberry Pi Raspbian Jessie with Ansible

At [Carl Group](http://www.carl-group.de/en/home/) we are using a small flock of Raspberry Pis for conference-branded screens, like schedule, voting results or social media walls with [SENDONSCREEN](http://send.on-screen.info). The Pis display dynamic web pages in full screen FullHD kiosk mode. They are distributed over the entire conference area and operate fully stand alone and have no keyboard or mouse attached.

The new [Raspbian Jessie Pixel desktop](https://www.raspberrypi.org/downloads/raspbian/) with Chromium on the Pi 3 works great. The new Pixel desktop with lightdm is really fast, responsive, lightweight and good looking. And it delivers Chromium out of the box. Great work by the Raspbian team!

## Ansible automation

Manually securing and configuring a Raspberry Pi for kiosk mode is boring and error prone. I love [Ansible](https://www.ansible.com/), so we are using Ansible to automate the installation and distribution of our Pi flock. The base image is Raspbian Jessie Pixel, which we copy to 8 GB SD cards.

Using the following playbooks is very easy. You will have your Pis running in a few minutes. And with a single command you will then be able to change the Chromium kiosk screen on all machines.
This is all tested on Raspbian Jessie on multiple Raspberry Pi 2 and 3s.

These playbooks use a local ansible.cfg file, which sets the hosts file.

Thanks @chhantyal for [5Minutes - Server Security Essentials](https://github.com/chhantyal/5minutes), which this repo is forked from.

### Basic requirements

- Ansible
- nmap network scanner (optional)
- Micro SD card reader/writer
- No Wifi setup, your Pis must be connected via ethernet cable to your network

### 1. Setup SD card image

Burn the Raspbian Jessie image on to your SD card. I use [Etcher](https://etcher.io/) for this.
Connect your Pi to the network via ethernet and boot it.

### 2. Find the IP adresses of your Pis and change Ansible hosts file

I use nmap on my management machine for finding out the IP addresses of my Pis.

```
$ sudo nmap -p 22 --open -sV 192.168.100.0/24
```

Set the Pi addresses in your Ansible hosts file.
Add the host variables pi_name and chrome_url. They are used by the playbooks for setting the hostname and the Chromium URL.

```
[pis]
192.168.100.67 pi_name=station1 chrome_url=http://kiosk1
192.168.100.68 pi_name=station2 chrome_url=http://kiosk2
192.168.100.71 pi_name=station3 chrome_url=http://kiosk3
```

### 3. Set more variables

Please check the variables in vars.yml, which you might want to change. You should at least set the Ansible management user and the key.

```
server_user_name: co
server_user_password: raspberry
user_public_keys:
  - ~/.ssh/id_rsa.pub
```

### 4. First run, bootstrap and secure your Pis

Use the user "pi" with the default password "raspberry".

You may want to check that your Pis are responding on the ssh port:

```
ansible all -m ping -u pi -k
```

Execute the secure.yml playbook to secure your Pis:
- sets hostname and timezone
- creates a new sudo user with key login
- disables ssh root access and password authentication
- installs some packages
- sets up ufw firewall

```
ansible-playbook secure.yml -u pi -k
```

After running the secure.yml playbook you can omit the username for Ansible and you won't be able to login with a password.

### 5. Update Raspbian (optional)

Use the apt.yml playbook to update the OS:

```
ansible-playbook apt.yml
```

This does only a safe upgrade, no dist upgrade. The playbook checks if the server needs to be restarted.

### 6. Setup Chromium kiosk

This is the main part. Use the chrome.yml playbook to set up Chromium in kiosk mode and make it autostart:

```
ansible-playbook chrome.yml
```

This does the following:
- set HDMI modes
- install unclutter and MS core fonts
- disable screensaver
- configure Chromium autostart

The playbook checks if the lightdm X environment needs to be restarted if you change the chrome_url variable.

### 7. Get info (optional)

Use the info.yml playbook to get some info about your Pi flock:

```
ansible-playbook info.yml
```

## Some feature ideas

- When using Wifi configure Wifi settings via Ansible
- Create the Ansible hosts file automatically from the output of the nmap scan
- Set the default language (accept language header) in Chromium
- Make the Pi 3 operate as a Bluetooth Beacon (iBeacon and/or Eddystone) device and notify smartphone users of nearby conference actions, configure Beacon settings via Ansible

## Contributing

I would like to hear from you. Comments, ideas, questions, tips, tests and pull requests are welcome.

## License

See [MIT License](LICENSE.txt)

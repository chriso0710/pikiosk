# Automate Chromium in kiosk mode and Eddystone beacon on Raspberry Pi Raspbian Jessie with Ansible

At [Carl Group](http://www.carl-group.de/en/home/) we are using a small flock of Raspberry Pis for conference-branded screens, like schedule, voting or social media walls with [SENDONSCREEN](http://send.on-screen.info). The Pis display dynamic web pages in full screen FullHD kiosk mode. Additionally they may work as Eddystone beacons and advertise Eddystone URLs.
The Pis are distributed over the entire conference area and operate fully standalone and have no keyboard or mouse attached.

The new [Raspbian Jessie Pixel desktop](https://www.raspberrypi.org/downloads/raspbian/) with Chromium on the Pi 3 works great. The new Pixel desktop with lightdm is really fast, responsive, lightweight and good looking. And it delivers Chromium out of the box. Great work by the Raspbian team!

The Pi 3 has Bluetooth builtin, which enables it to work as a BT beacon.

## Ansible automation

Manually securing and configuring a Raspberry Pi for kiosk mode is boring and error prone. I love [Ansible](https://www.ansible.com/), so we are using Ansible to automate the installation and distribution of our Pi flock. The base image is Raspbian Jessie Pixel, which we copy to 8 GB SD cards.

Using the following playbooks is very easy. You will have your Pis running in a few minutes. And with a single command you will then be able to change the Chromium kiosk screen or Eddystone beacon on all machines.
This is all tested on Raspbian Jessie on multiple Raspberry Pi 2 and 3s.

These playbooks use a local ansible.cfg file, which sets the hosts file.

Thanks @chhantyal for [5Minutes - Server Security Essentials](https://github.com/chhantyal/5minutes), which this repo is forked from.

### Basic requirements

- Raspbian full Jessie with Pixel (not Jessie-lite)
- Ansible
- nmap network scanner (optional)
- Micro SD card reader/writer

### 1. Setup SD card image

Burn the Raspbian Jessie image on to your SD card. I use [Etcher](https://etcher.io/) for this.
Connect your Pi to the network via ethernet cable, power it on and boot.

If you create a valid `wpa_supplicant.conf` file and place in your boot partition the Pi copies it over using stock images (like Raspbian Jessie). The file disappears after it copies, but activates network on first boot so you should be able to find your Pi via nmap/ssh in your Wifi network.

### 2. Find the IP adresses of your Pis and change Ansible hosts file

I use nmap on my management machine for finding out the IP addresses of my Pis.

```
$ sudo nmap -p 22 --open -sV 192.168.100.0/24
```

Set the Pi addresses in your Ansible hosts file.
Add the host variables `pi_name`, `chrome_url` and `beacon_url`. They are used by the playbooks for setting the hostname and the Chromium and Eddystone URLs.

```
[pis]
192.168.100.67 pi_name=station1 chrome_url=http://kiosk1
192.168.100.68 pi_name=station2 chrome_url=http://kiosk2
192.168.100.71 pi_name=stationx chrome_url=http://kiosk3 beacon_url=https://goo.gl/xyz
```

### 3. Set more variables

Please check the variables in vars.yml, which you might want to change. You should at least set the Ansible management user and the key.

```
server_user_name: kiosk-admin
server_user_password: raspberry
user_public_keys:
  - ~/.ssh/id_rsa.pub
```

### 4. First run, bootstrap and secure your Pis

Use the user "pi" with the default password "raspberry".

You may want to check that your Pis are responding on the ssh port:

```
$ ansible all -m ping -u pi -k
```

Execute the secure.yml playbook to secure your Pis:
- sets hostname and timezone
- creates a new sudo user with key login
- disables ssh root access and password authentication
- installs some packages
- sets up ufw firewall

```
$ ansible-playbook secure.yml -u pi -k
```

After running the secure.yml playbook you can omit the username for Ansible and you won't be able to login with a password.

### 5. Update Raspbian (optional)

Use the apt.yml playbook to update the OS:

```
$ ansible-playbook apt.yml
```

This does only a safe upgrade, no dist upgrade. The playbook checks if the server needs to be restarted.

### 6. Setup Chromium kiosk

This is the main part. Use the chrome.yml playbook to set up Chromium in kiosk mode and make it autostart:

```
$ ansible-playbook chrome.yml
```

This does the following:
- set HDMI modes
- install unclutter and MS core fonts
- disable screensaver
- configure Chromium autostart

The playbook checks if the lightdm X environment needs to be restarted if you changed the chrome_url variable.
After changing HDMI modes the Pi will reboot.

### 7. Get info (optional)

Use the info.yml playbook to get some info about your Pi flock:

```
$ ansible-playbook info.yml
```

### 8. Setup Eddystone beacon (optional, only on Pi 3)

Google's Eddystone protocol can advertise a URL, which makes it convenient and easy to use. We use bluetooth beacons for voting in events and conferences.
iOS and Android scan and show surrounding Eddystone beacons with special apps or Chrome on Android. Use the variable `beacon_url` in the inventory to set the URL.

```
$ ansible-playbook eddystone.yml
```

Please note:
- Eddystone URL length is limited, so you might want to use a URL shortener like http://bit.ly or http://goo.gl
- Beacon advertising does not survive a Pi reboot

## Troubleshooting

### Connection Rejected Error

If you cannot access the Pi after Step 4 it's likely you may be using an incorrect user or private key.
Whilst you can link to a private-key file using command line arguments with

```
$ ansible-playbook -u <user> --private-key=<path-to-private-key>
```

it's probably easier all round if you edit the `ansible.cfg` file like so

```
cat >> ansible.cfg << ANSIBLE_CFG
private_key_file = ~/.ssh/2016_4096_rsa
remote_user = kiosk-admin
ANSIBLE_CFG
```

It should be noted in the example given `kiosk-admin` is the variable `server_user_name` to administer the pi and `~/.ssh/2016_4096_rsa` is the private custom SSH key, which must correspond to the public key in variable `user_public_keys` in the `vars.yml` file.

## Some feature ideas

- When using Wifi configure Wifi settings via Ansible
- Create the Ansible hosts file automatically from the output of the nmap scan
- Set the default language (accept language header) in Chromium
- Splash screen

## Technologies

- [Ansible](https://github.com/ansible/ansible), tested with 2.2
- [PyBeacon](https://github.com/nirmankarta/PyBeacon)

## Contributing

I would like to hear from you. Comments, ideas, questions, tips, tests and pull requests are welcome.

## License

See [MIT License](LICENSE.txt)

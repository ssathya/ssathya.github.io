
# Installing open VPN in an VM
The following instructions are based on this [Youtube video](https://www.youtube.com/watch?v=vU0jyDGGUUE)
## Introduction
When I want to virtually travel to a different country I need to create a VPN server in that region and I often land up watching Youtube videos or installing the software wrong. So I’m creating this gist so that I will not be lost when I install my next OpenVPN server on a VM.

## Platform
Though we can use any VM, I prefer using LightSail offering from AWS to running my OpenVPN. Necessary changes need to be done (only in IP Firewall settings) based on the selected platform. I usually select the smallest server, (512  MB RAM, 1  vCPU, 20  GB  SSD) to run my OpenVPN as it is going to **serve only one client *me***.

## Prepare the server
The first change I do on any server is to edit the *.bashrc* file. I had the following two lines at the bottom of the file
```sh
PATH=./:~/bin/:$PATH
set -o vi
```
Log out and log in to the server to load the changes to the environment.
### System Update
Let us bring the latest updates to the operating system; I, almost always, use Ubuntu and I run the following commands:
```sh
sudo apt update && sudo apt upgrade -y
sudo reboot now
```
Of course, Ubuntu will ask you a couple of questions during the update and reboot requests; answer accordingly. 
Once the server reboots:
```sh
sudo do-release-upgrade
sudo reboot now
```
As usual, the OS will ask for your confirmations every now and then; answer them to ensure the installation completes.
*It depends on the server location and available resources but until now you should have spent around 30 to 45 minutes bringing up the server!*
### Start installing packages
It's time to install the necessary packages; I like to use *nala* instead of *apt* so I start with installing **nala**.
```sh
sudo apt install nala
```
Next, let us install docker:
```sh
nala install docker.io
```
After installing docker the system might request you reboot but let us issue one more command and then reboot the server; we'll add the current user to the user group docker and then reboot the server.

```sh
sudo usermod -aG docker $USER
sudo reboot now
```
While the server is rebooting we need to open port 1194 so that your PC can communicate with the VPN server. Select your server in the Lightsail console as shown in the following image. In the following image, the name of my Server is OpenVPN and I'll click on it.
![Select server first](https://i.imgur.com/ReVi1qa.png)

You will be presented with your server's console and on this screen click on the Network menu item:
![Now select Networking menu item](https://i.imgur.com/aPFuTCe.png)
Update firewall rules so that we can use port 1194 using UDP protocol. In the image below I'm opening ports 1190 to 1199 (don't ask me why; not necessary but I like to do it this way).
![Update firewall to allow port 1190 to 1199](https://i.imgur.com/Q3URhGh.png)

# I've updated this document and the changes are at the end with topic "Updates". It is much easier than the method mentioned below.

### Install OpenVPN and configure it
Your server should have rebooted by now; if not time for a coffee break.  Okay, let's roll up our sleeves!

Hold on! Not so fast. Let's confirm we are ready.
```sh
docker run hello-world
```
Make sure we get no errors and now let us prepare to use openVPN.
```sh
docker pull kylemanna/openvpn
```
This should pull the necessary image to run openvpn. 
Make a note of the servers' public IP. We'll be using it in one of the following commands.
To keep all VPN content in one folder create a folder, say openVPN, and cd to that folder: now let's run the following commands:
```sh
# Create and initialize openvpn
docker run --rm -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_genconfig -u udp://<YOUR PUBLIC IP>

# Create certificate
docker run --rm -v $PWD:/etc/openvpn -it kylemanna/openvpn ovpn_initpki
# it might ask you to provide phrases to encrypt the key; use a strong key and save it; you'll need it later.

# Create a client account
docker run --rm -v $PWD:/etc/openvpn -it kylemanna/openvpn easyrsa build-client-full <User ID>

# Copy client certificate (ovpn file) from container
docker run --rm -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_getclient <User ID> > <User ID>.ovpn
```
At this stage, we should have a file <User ID>.ovpn in the openVPN folder. Transfer this file to your client's machine.
Finally, start the Open VPN server:
```sh
docker run --name openvpn -v $PWD:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN --restart always kylemanna/openvpn
# Verify it is running
docker ps
```
All done; check if you can use the VPN server!

  ## Updates
  
  Once you've installed docker, let us use a different docker image, which automates most of the manual work mentioned above. Install the  **alekslitvinenk/openvpn**  as follows:
  ```sh
  docker run -itd --rm --cap-add=NET_ADMIN \
-p 1194:1194/udp -p 80:8080/tcp \
-e HOST_ADDR=$(curl -s https://api.ipify.org) \
--name dockovpn alekslitvinenk/openvpn
  ```
  Make sure the VM has port 80 open for HTTP (default on with LightSail) and 1194 (details above) open. On your laptop/desktop, navigate to http://*Your servers IP address*, and your client profile will be downloaded to your local machine (the *ovpn* file). Use the profile to connect to your VPN server as mentioned above.
### Security
Anyone accessing your IP address can download the ovpn file and use your VPN server. So, for security, it would be better to shut down the server or block port 80 when not in use.
  

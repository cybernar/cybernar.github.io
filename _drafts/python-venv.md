# Configuration de Python et GDAL dans Ubuntu

Python3 est déjà installé dans Ubuntu. Pour installer un groupe de package pour une utilisation bien ciblée (exemple faire tourner Fototex), il est judicieux d'utiliser des environnements virtuels python avec venv. On peut aussi utiliser conda, mais voici un exemple avec venv.

## Exemple : installation de Fototex dans un environnement virtuel venv


sudo apt install gnupg software-properties-common
wget -qO - https://qgis.org/downloads/qgis-2021.gpg.key | sudo gpg --no-default-keyring --keyring gnupg-ring:/etc/apt/tr
usted.gpg.d/qgis-archive.gpg --import
sudo chmod a+r /etc/apt/trusted.gpg.d/qgis-archive.gpg
sudo add-apt-repository "deb https://qgis.org/ubuntu-ltr $(lsb_release -c -s) main"
sudo apt build-essential libgdal-dev gdal-bin
gdalinfo --version

python3 --version

sudo apt install python3-venv build-essential python3-dev 

mkdir ~/bacasable
python3 -m venv /home/bernard/bacasable/fototest/env
cd ~/bacasable/fototest
source env_fototest/bin/activate

deactivate

sudo apt install python3-tk scikit-image
pip install numpy
pip install matplotlib
pip install scikit-learn
pip install numba
pip install h5py
pip install wheel
pip install tk
pip install fototex

easy_install GDAL=3.0.4

Installer le serveur X sur  Windows 10

Now with WSL 2 installed, we can download and install VcXsrv. 
https://sourceforge.net/projects/vcxsrv/
In my opinion, seems to the best choice for X-Server in Windows

Config : WindowMode="Multiple Window" Display number="0" Clipboard="True" Clipboard Primary="True" Native opengl="True" Disable Access Control="True" 

-> config xlaunch

Ceci fonctionne :

https://stackoverflow.com/questions/61860208/wsl-2-run-graphical-linux-desktop-applications-from-windows-10-bash-shell-erro

Lire aussi : 

https://stackoverflow.com/questions/43397162/show-matplotlib-plots-and-other-gui-in-ubuntu-wsl1-wsl2
https://stackoverflow.com/questions/61110603/how-to-set-up-working-x11-forwarding-on-wsl2


Démarrer le serveur X :

export DISPLAY=$(route.exe print | grep 0.0.0.0 | head -1 | awk '{print $4}'):0.0
glxgears

sudo apt install -y mesa-utils

Visual Studio Code

code .

Partage de fichiers

Depuis Ubuntu :
ls /mnt/c/Users/BernardC/Documents/

Depuis Windows : 
\\wsl$
\\wsl$\Ubuntu-20.04\home\bernard

## Désinstaller Ubuntu

    wsl --unregister <distroName>

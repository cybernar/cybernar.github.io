# Installer WSL2 sur Windows 10

## Résumé (pour ceux qui n'ont pas le temps de tout lire).

**WSL** alias **Windows Subsystem for Linux** est une fonctionnalité développée par Microsoft sur Windows 10 et 11 et qui permet d'avoir une console Linux dans Windows.

Quels sont les avantages ?

- facile et rapide à installer (il faut avoir les droits administrateurs cependant !)
- on peut choisir la distribution : Ubuntu, Debian, OpenSuse ... on peut même avoir plusieurs distros sur 1 même poste
- très rapide à démarrer (plus rapide qu'une machine sous Virtual Box en tout cas !)
- très facile d'accéder aux fichiers Windows depuis Linux, et vice-versa
- une fois la distribution installée, on peut installer et on met à jour les paquets avec `apt` (sur Debian et Ubuntu)

Pour quoi faire ?

- idéal quand on veut tester un script Python à la fois sur Linux et Windows ! 
- idéal quand on a des scripts shell à exécuter
- WSL s'intégre très bien dans l'IDE **Visual Studio Code** : cela veut dire qu'on peut exécuter des codes sur Linux alors que l'IDE tourne sur Windows

Quels sont les (petits) inconvénients ?

- vous utilisez Linux en mode console : pas d'interface graphique par défaut, donc pas de possibité de visualiser des données (exemple Matplotlib). Il est cependant possible d'installer un **serveur X** pour pallier à ce problème, mais c'est un peu technique sur Windows 10.
- WSL, tout comme Docker pour Windows, dépend de Hyper-V. Or Hyper-V est incompatible avec VirtualBox. Donc pour utiliser VirtualBox il faut désactiver Hyper-V, et pour utiliser WSL il faut le réactiver !

## Page officielle de WSL

<https://docs.microsoft.com/fr-fr/windows/wsl/install>

## Installation d'une distribution Linux avec WSL

Ouvrir **Windows Powershell** en administrateur (choisir **Exécuter en tant qu'administrateur**)

Pour voir la liste des distributions Linux disponibles, tapez :

    wsl --list --online

Pour installer une distribution, tapez :

    wsl --install -d <NAME>

Par exemple pour installer Ubuntu 20.04 LTS, tapez :

    wsl --install -d Ubuntu-20.04

Remarque : la distribution par défaut (quand on ne précise pas le nom avec l'option -d) est Ubuntu, en théorie la plus récente version ... mais en réalité ce n'est pas forcément la toute dernière release qu'on obtient.

## Post-installation

**Important :** juste après l'installation, au 1er lancement d'Ubuntu, vous devez définir le login et le mot de passe de l'utilisateur. Notez bien ce mot de passe car il sera demandé pour la commande `sudo`.

Vous venez d'installer Ubuntu : il va se lancer automatiquement.

Définissez le nom et le mot de passe. Exemple : espace-dev / mdp2022

## Lancer Ubuntu

Allez dans le menu Démarrer de Windows et dans la liste des programmes cherchez **Ubuntu**.

## Mettre à jour Ubuntu

Dans la console tapez :

    sudo apt update
    sudo apt upgrade

## Partage de fichier entre Windows 10 et Linux Ubuntu

### Depuis Ubuntu, dans la console

    ls /mnt/c/Users/BernardC/Documents/

### Depuis Windows, dans l'explorateur

Pour voir les fichiers de toutes les distros :

    \\wsl$

Par exemple, accéder à son répertoire `home` ds la distro Ubuntu 20.04 :

    \\wsl$\Ubuntu-20.04\home\espace-dev

## Configuration de Python et GDAL dans Ubuntu

Python3 est déjà installé dans Ubuntu. Pour installer un groupe de package pour une utilisation bien ciblée (exemple faire tourner Fototex), il est judicieux d'utiliser des environnements virtuels python avec venv. On peut aussi utiliser conda, mais voici un exemple avec venv.

### Exemple : installation de Fototex dans un environnement virtuel venv


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

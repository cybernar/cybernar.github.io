---
layout: post
title: "Installer un environnement de développement Python avec Conda"
date: 2022-01-27
---

Pour traiter des données satellitaires, j'ai besoin d'installer un environnement de développement Python avec des packages tels que GDAL, rasterio, xarray. Comment procéder ? 

Certains systèmes (Linux notamment) offrent un Python déjà installé, mais en qui concerne les packages, les dépendances sont tellement difficiles à gérer qu'il vaut mieux utiliser des **environnements virtuels**. Cela permet d'avoir plusieurs installations de Python en parallèle, de passer de l'une à autre facilement, d'exporter une configuration d'un système vers un système, de supprimer facilement des environnements qui ne servent plus.

Que je sois sur Windows, Mac ou même Linux, 2 options s'offrent à moi.

- VirtualEnv
- Conda

Ici je récapitule tout ce que j'ai appris sur **conda** qui semble plus indiqué pour une utilisation "data science" (notamment si on veut utiliser et créer des notebooks).

## Cheatsheet conda

Dernière version : <https://docs.conda.io/projects/conda/en/latest/_downloads/cb0ffc4c7b1e6c0e716c066d2b077faf/conda-4.12.pdf>

## Anaconda ou Miniconda ?

- [Miniconda](https://docs.conda.io/en/latest/miniconda.html) est un gestionnaire d'environnements qui vous permettra d'installer Python et les packages dans un environnement cloisonné. C'est une version minimaliste de conda qui fonctionne en lignes de commande.
- [Anaconda](https://anaconda.cloud/installers) est une version interface graphique de conda, orientée "analyse de données". Elle inclut des environnements de développement tels que Spyder, Jupyter.

Personnellement je trouve l'interface d'Anaconda assez lente et frustante, j'ai donc fini par l'abandonner et adopter Miniconda sur ttes mes machines. Si vous n'êtes pas fâché avec la ligne de commande, autant utiliser Miniconda.

## Installation de Miniconda sur Linux et Mac

Voir <https://docs.conda.io/en/latest/miniconda.html>.

Que ce soit sur Windows, Mac ou Linux, **Miniconda** installera un environnement par défaut qui s'appelle **base**. 

Sur Windows, vous aurez dans le menu des Programmes une nouvelle entrée *Anaconda 3* dans laquelle 2 menus Powershell, ou Invite de Commande, au choix, vous permettent d'accéder à cet environnement **base**.

Sur Mac ou Linux, c'est un peu différent : vous ouvrez le Terminal, puis vous activer l'environnement avec :

    conda activate base 

Il est possible de faire en sorte que l'environnement base soit activé dès l'ouverture du Terminal mais je n'aime pas trop, je préfère pour ma part que conda soit activé quand je le demande uniquement. C'est pourquoi à la fin de l'installation sur Linux ou Mac, ce message s'affiche :

    You have chosen to not have conda modify your shell scripts at all.
    To activate conda's base environment in your current shell session:

    eval "$(/home/bernard/Apps/anaconda3/bin/conda shell.YOUR_SHELL_NAME hook)" 

    To install conda's shell functions for easier access, first activate, then:

    conda init

    If you'd prefer that conda's base environment not be activated on startup, 
    set the auto_activate_base parameter to false: 

    conda config --set auto_activate_base false

    Thank you for installing Anaconda3!

## Création d'un environnement

C'est parti, vous pouvez créer de nouveaux environnements avec les versions de Python que vous souhaitez.

1. Création d'un environnement geo_env avec Python 3.8 et le package mamba

    conda create -n geo_env -c conda-forge python=3.8 mamba

(pourquoi mamba ? voir ci-dessous)

2. Activation du dépôt geo_env

    conda activate geo_env

3. Lister les environnements

`conda info --envs`

`conda env list` (déprécié)

## Mise à jour de conda

MAJ conda : `conda update -n base -c defaults conda`

MAJ tous les packages (déconseillé car c'est long) : `conda update --all`

## Gestion des dépôts (ou channels)

Pour installer un (des) package(s) pkg1, pkg2, ... il suffit de taper `conda install pkg1 pkg2`, et le package est téléchargé sur internet, mais en fait c'est un peu plus complexe. D'abord un package peut dépendre d'autre qui vont être installés à leur tour. Les packages ont des versions, les packages dépendants aussi. conda doit résoudre les dépendances.

Les packages sont disponibles dans des dépôts appelés **channels** ... et certains dépôts proposent des versions plus récentes.

L'utilisation du dépôt `conda-forge` est fortement recommandée afin d'obtenir la dernière version des packages. Pour exemple, pour installer **rasterio** sur Python 3.8, il faut installer la dernière version du package et pour cela, passer par conda-forge.

### Commandes pour la configuration des channels

cf. <https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-channels.html> et <https://docs.conda.io/projects/conda/en/latest/commands/config.html>

Voir quels sont les channels ?

    conda config --show-sources

Ajouter un channel en top of the list

    conda config --add channels conda-forge

Changer la priorité pour les versions / canaux

    conda config --describe channel_priority
    conda config --set channel_priority false

### Utilisation de mamba

Enfin, l'utilisation de la commande `mamba` permet de gagner beaucoup de temps dans l'installation des packages, car elle accélère la résolution des dépendances.

1. Création d'un environnement geo_env avec Python 3.8 et mamba

    conda create -n geo_env -c conda-forge python=3.8 mamba

2. Activation du dépôt geo_env

    conda activate geo_env

3. Installation des packages numpy, pandas, gdal, rasterio, geopandas et autres

    mamba install numpy scipy matplotlib seaborn pandas openpyxl numba
    mamba install jupyterlab
    mamba install shapely fiona gdal rasterio geopandas rasterstats sentinelsat rtree


## Save environments

Sauvegarde la liste exhaustive : 

    conda env export > gisenv.yml

Sauvegarde uniquement la liste des packages explicitement installés : 

    conda env export --from-history > gisenv-history.yml

## Autre exemple (sans mamba) : création d'un environnement GISENV

conda create --name gisenv python=3.9
conda activate gisenv
conda install -c defaults pandas seaborn
conda install -c conda-forge rasterio
conda install -c conda-forge shapely
conda install -c conda-forge gdal # gdal 3.3 
conda install -c conda-forge pandas openpyxl geopandas
conda install -c conda-forge spyder

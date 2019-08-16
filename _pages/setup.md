---
layout: archive
title: "Before you come"
permalink: /setup/
author_profile: false
header:
  image: /assets/images/download.png
  caption: "Photo credit: [*D3.js*](https://observablehq.com/@mbostock/hover-voronoi)"
---


All software for the course can be installed without having administrator
privileges on your computer.  We achieve this by using a package manager called
[Miniconda](https://docs.conda.io/en/latest/miniconda.html). Miniconda creates isolated environments where software packages and package dependencies can be installed from a small file listing the required packages.
{: .notice--info}


# Windows 10 instructions
(modified from Nicholas Dawson's [instructions](https://atmoguy.com/python-for-scientists-install-windows/))

## Download
Go here to download the latest version of Miniconda. Download Miniconda3, not Miniconda2! Download either the 32-bit or 64-bit version that corresponds to your computers architecture. If you don’t know which architecture you’re using, check [this](https://www.howtogeek.com/howto/21726/how-do-i-know-if-im-running-32-bit-or-64-bit-windows-answers/) out. I’m going to download the 64-bit version of Miniconda3.

![Anaconda download page](/ml-training-site/assets/images/Screenshot_4.png)

## Install locally

Open the downloaded Anaconda installer and select the installation type `Just Me (recommended)`.


![Anaconda install page 1](/ml-training-site//assets/images/Screenshot_1.png)


## Choose the install location

Choose the default install location if it is available. that should be

```
C:\Users\{username}\AppData\Local\Continuum\miniconda3
```

![Anaconda install page 2](/ml-training-site//assets/images/Screenshot_2.png)

##  Choose advanced options

Select the default advanced options.

![Anaconda install page 3](/ml-training-site//assets/images/Screenshot_3.png)

## Post install

Your miniconda shell should now be available under the start menu

![Anaconda install page 5](/ml-training-site//assets/images/Screenshot_5.png)

## Create a conda environment for the course

A few days before the course we will post a conda `environment.xml` file here.

Once that file is available you can create an environment and install the required packages by running the command:

```
conda env create --f environment.xml
```


For more info refer to the [conda docs](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)

Once it has been created activate the environment using

```
conda activate mlenv
```

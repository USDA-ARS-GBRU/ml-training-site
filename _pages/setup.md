---
layout: single
toc: true
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
Go [here](https://docs.conda.io/en/latest/miniconda.html) to download the latest version of Miniconda. Download Miniconda for Python 3.7 (not2.7) Download either the 32-bit or 64-bit Windows version that corresponds to your computer's architecture. If you don’t know which architecture you’re using, check [this](https://www.howtogeek.com/howto/21726/how-do-i-know-if-im-running-32-bit-or-64-bit-windows-answers/) out. Most users will download the 64-bit version.

![Anaconda download page](/ml-training-site/assets/images/Screenshot_4.png)

## Install locally

Open the downloaded Miniconda installer and select the installation type `Just Me (recommended)`.


![Anaconda install page 1](/ml-training-site//assets/images/Screenshot_1.png)


## Choose the install location

Choose the default install location if it is available. That should be:

```
C:\Users\{username}\AppData\Local\Continuum\miniconda3
```

![Anaconda install page 2](/ml-training-site//assets/images/Screenshot_2.png)

##  Choose advanced options

Select the default advanced options.

![Anaconda install page 3](/ml-training-site//assets/images/Screenshot_3.png)

## Post install

Your Miniconda shell should now be available under the start menu

![Anaconda install page 5](/ml-training-site//assets/images/Screenshot_5.png)

## Create your conda environment

A few days before the course we will post a conda `environment.xml` file here.

Once that file is available you can create an environment and install the required packages by running the command:

```
conda env create --f environment.xml
```


For more information refer to the [conda docs](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)

Once your conda environment has been created activate the environment using

```
conda activate mlenv
```







# MacOS 10.14 instructions

## Download
Go [here](https://docs.conda.io/en/latest/miniconda.html) to download the
latest version of Miniconda. Download for Python 3.7
(not 2.7). Download the `Mac OS X 64-bit (.pkg installer)`

![Anaconda download page](/ml-training-site/assets/images/mac1.png)

## Install locally

Open the downloaded Anaconda installer and install in the default location

![Anaconda install page](/ml-training-site//assets/images/mac2.png)


## Post install

You can now open up your mac's Terminal program under `Applications -> Utilities -> Terminal.app`. If you see `(base)`at the beginning of your Terminal prompt then conda was installed successfully.

![Terminal](/ml-training-site//assets/images/mac3.png)

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

# Linux instructions

## Download
Go [here](https://docs.conda.io/en/latest/miniconda.html) to download the
latest version of Miniconda. Download for Python 3.7. Download either the
32-bit or 64-bit Windows version that corresponds to your computer's
architecture. If you don’t know which architecture you’re using type `arch`
into the terminal. Download the `Linux 64-bit (bash installer)` or the `Linux 64-bit (bash installer)`.

![Anaconda download page](/ml-training-site/assets/images/ubuntu1.png)

## Install locally

Open the terminal and move into the directory where Miniconda was downloaded and
run the Bash script to install Miniconda

```
bash Miniconda3-latest-Linux-x86_64.sh
```

## Post install

Close and reopen your Terminal.  If you see `(base)`at the beginning of your Terminal prompt then conda was installed successfully.


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

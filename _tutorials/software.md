---
title: "Software to install before the microbiome workshop"
excerpt: "Instructions for installing the required tutorial software on Windows, Mac and Ubuntu machines"
layout: single
author: "Adam Rivers"

---
{% include toc %}
Most of the tutorials that will be done at the microbiome workshop will be done remotely on Ceres, the high-performance computer, but some basic software is needed to connect and visualize the data we will be generating.  If you do not have administrator rights on your computer you may need to ask a departmental IT specialist to install Docker for you before the meeting. All other programs should be usable without administrator rights. Instructions for major operating systems are listed below.


# Windows:

1. Putty
  * A terminal client for connecting to Ceres (or chrome can be used for SSH with limited features)
  * Available here: [http://www.putty.org/](http://www.putty.org/)
2. Google Chrome Browser
  * Needed for visualizing Qiime2 ad Anvi'o data efficiently
  * Available here: [https://www.google.com/chrome/](https://www.google.com/chrome/)
3. Bandage for viewing FASTG files
  * Available here: [https://rrwick.github.io/Bandage/](https://rrwick.github.io/Bandage/)
4. Docker (requires administrator rights)
  * Docker is a lightweight software container manager needed to run Anvi'o
  * Available here: [https://www.docker.com/docker-windows](https://www.docker.com/docker-windows)
5. Anvi'o Docker image
  * Once Docker has been installed Anvi'o and all of its dependencies can be installed using Docker.

### Anvi'o installation

Open a terminal program like Putty and pull a docker image from Docker Hub with this command
```bash
docker run -p 8080:8080 -it meren/anvio:latest
```
Docker will download the images and start a container.

You should now see a funny cursor that looks like this telling you you are in a container:

```
 :: anvi'o ::  / >>>
```

You can test your installation using the following command:
```
:: anvi`o ::  / >>> anvi-self-test
```
Open the Chrome browser enter ```localhost:8080``` in the address bar and press enter. A web application for visualizing the test data will appear.

# Mac:
1. Terminal
  * built in
2. Google Chrome Browser
  * Needed for visualizing Qiime2 ad Anvi'o data efficiently
  * Available here: [https://www.google.com/chrome/](https://www.google.com/chrome/)
3. Bandage for viewing FASTG files
  * Available here: [https://rrwick.github.io/Bandage/](https://rrwick.github.io/Bandage/)
4. Docker (requires administrator rights)
  * Docker is a lightweight software container manager needed to run Anvi'o
  * Available here: [https://www.docker.com/docker-mac](https://www.docker.com/docker-mac)
5. Anvi'o Docker image
  * Once Docker has been installed Anvi'o and all of its dependencies can be installed using Docker.

### Anvi'o installation
Open Terminal and pull a docker image from Docker Hub with this command

```bash
docker run -p 8080:8080 -it meren/anvio:latest
```
Docker will download the images and start a container.

You should now see a funny cursor that looks like this telling you you are in a container:

```
:: anvi'o ::  / >>>
```

You can test your installation using the following command:
```
:: anvi`o ::  / >>> anvi-self-test
```
Open the Chrome browser enter ```localhost:8080``` in the address bar and press enter. A web application for visualizing the test data will appear.

# Ubuntu/Debian Linux:
1. Terminal
  * built in
2. Google Chrome Browser
  * Needed for visualizing Qiime2 ad Anvi'o data efficiently
  * Available here: [https://www.google.com/chrome/](https://www.google.com/chrome/)
3. Bandage for viewing FASTG files
  * Available here: [https://rrwick.github.io/Bandage/](https://rrwick.github.io/Bandage/)
3. Docker and Anvi'o (requires administrator rights)
  * Docker is a lightweight software container manager needed to run Anvi'o
  * Anvi'o is metagenomic analysis and visualization software

### Docker and Anvi'o installation
With Linux you can install docker with a package manager like apt-get

```bash
sudo apt-get install docker.io
```

Start up docker with this command:
```bash
sudo service docker.io start
```
Open Terminal and pull a docker image from Docker Hub with this command
```bash
sudo docker pull meren/anvio
```
Run the latest image
```bash
sudo docker run --rm -it meren/anvio:latest
```
You should now see a funny cursor that looks like this telling you you are in a container:

```
 :: anvi'o ::  / >>>
```

You can test your installation using the following command:
```
:: anvi`o ::  / >>> anvi-self-test
```
Open the Chrome browser enter ```localhost:8080``` in the address bar and press enter. A web application for visualizing the test data will appear.

tiledev
=======
Notes on generating, storing, and serving tiles.

# Environment
The development and testing below was performed in EC2 using the Quick Start Ubuntu AMI - Ubuntu Server 14.04 LTS (HVM). This AMI already comes with Python, so the following were additional commands run to get started:

**Common dependencies**
```shell
sudo apt-get update
sudo apt-get install python-pip
sudo apt-get install gdal-bin
sudo apt-get install python-gdal
sudo pip install awscli
```

**Additional dependencies for WMS caching**
```shell
sudo aptitude install build-essential python-dev libjpeg-dev zlib1g-dev libfreetype6-dev
sudo pip install Pillow
sudo pip install MapProxy
```

# Generating tiles
## ... from WMS
In many cases, a WMS may exist that is slow and/or unreliable. In this case, tiles may be generated from this existing service and cached for serving by alternate means. The primary tool involved is [MapProxy](http://mapproxy.org/) that can perform a number of tasks, but the primary one most relevant for caching tiles is the seeding functionality.

## ... from a source layer

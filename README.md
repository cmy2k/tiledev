tiledev
=======
Notes on generating, storing, and serving tiles.

# Environment configuration
The development and testing below was performed in EC2 using the Quick Start Ubuntu AMI - Ubuntu Server 14.04 LTS (HVM) on a t2 Micro instance. This AMI already comes with Python, so the following were additional commands run to get started:

**Common dependencies**
```shell
sudo apt-get update
sudo apt-get install python-pip
sudo apt-get install gdal-bin
sudo apt-get install python-gdal
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

# Storing tiles
## Output observations
Tiling will generate a _lot_ of files. Before proceeding, it is a Very Good Idea to consider what zoom levels are actually appropriate for serving the data, and any [good mapping library](http://leafletjs.com/) will have a way to define a maximum zoom level for a layer, but allow those tiles to be stretched beyond that point (in Leaflet, see maxZoom and maxNativeZoom). Leaflet by default supports zoom levels up to 19, which is about the scale of a few buildings.

As an experiment, tiles were generated for a [CONUS-scale MODIS NDVI product](http://forwarn.forestthreats.org/) with a resolution of 231m. The source file is around 60M and zoom levels were generated from 1-12 (12 is about the scale of a medium-sized city).

Some observations on the output:

| Z  | Time | Count    | Size |
|----|------|---------:|------|
| 12 | 86m  | 355,888  | 1.6G |
| 11 | 14m  | 89,586   | 567M |
| 10 | 08m  | 22,491   | 279M |
| 09 | 03m  | 5,700    | 93M  |
| 08 | 01m  | 1,440    | 27M  |
| 07 | ???  | 384      | 7.4M |
| 06 | ???  | 108      | 2.1M |
| 05 | ???  | 35       | 604K |
| 04 | ???  | 12       | 179K |
| 03 | ???  | 4        | 56K  |
| 02 | ???  | 2        | 24K  |
| 01 | ???  | 1        | 12K  |
|    |      |          |      |
| All |112m | 475,651  | 2.5G |

The growth is clearly exponential in processing time and output space. As a very rough napkin-math equation for conversation (but note that it isn't perfect since it underestimates what is observed):

nTiles = 0.135e<sup>1.1912(zLevels)</sup>

Assuming that each tile is 12K on disk, extrapolating these numbers out to 19 zoom levels yields the following:

| Z  | Count       | Size (GB) |
|----|------------:|-----------|
| 13 | 717,207     | 9         |
| 14 | 2,360,348   | 28        |
| 15 | 7,767,971   | 93        |
| 16 | 25,564,609  | 307       |
| 17 | 84,133,847  | 1,010     |
| 18 | 276,886,851 | 3,323     |
| 19 | 911,242,399 | 10,935    |
|    |             |           |
|1-19|1,309,148,881| 15,710    |

This is over 1.3 billion tiles and nearly 16 terabytes on disk. And remember that the equation likely vastly underestimates the actual output.

## Transport from EC2 to S3
The easiest tool that fits with the Python/PIP functionality above is the [awscli](http://aws.amazon.com/cli/).

```shell
sudo pip install awscli
```
After install, enter credentials (access key ID, secret access key) interactively with the command:
```shell
aws configure
```

Moving files to S3 then is as simple as:
```shell
aws s3 cp tile/output/path s3://bucketName/destination/path --recursive
```

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
## ... from a source layer
The installed gdal-bin comes with a utility called gdal2tiles. This converts an input data layer into a directory of output tiles. Its use is simple: 
```shell
gdal2tiles.py -z 1-10 inputLayer.extension tile/output/path
```

The -z flag and following argument specifies that zoom levels 1 to 10 will be generated. Should the process terminate and you want to generate only the missing tiles, use the -e flag.

Generally, the tiles will then be dumped in an output structure of /z/x/y

I have observed that clipping a styled image works best while providing an output alpha band. This avoids black areas around the data. For example:

```shell
gdalwarp -q -cutline boundaryMask.shp -crop_to_cutline -dstalpha -of GTiff input.tif output.tif
```

I have also observed that gdal2tiles complains if the projection is 4326 or the like. My best results came from reprojecting the file to 3857, and running it as such. The commands would be:

```shell
gdalwarp -t_srs EPSG:3857 a.tif b.tif
gdal2tiles.py -s EPSG:3857 -z 1-10 b.tif output/
```

Note: the output from gdal2tiles uses the TMS spec for naming tiles that reverses the oder of tiles. In order to have these appear correctly in Leaflet, the *tms* property must be set to true in the layer definition.

## ... from WMS
In many cases, a WMS may exist that is slow and/or unreliable. In this case, tiles may be generated from this existing service and cached for serving by alternate means. The primary tool involved is [MapProxy](http://mapproxy.org/) that can perform a number of tasks, but the primary one most relevant for caching tiles is the seeding functionality.

The seed command is used as follows:

```shell
mapproxy-seed -f mapproxy.yaml -s seed.yaml
```

There are a few properties that need to be defined for the seeding process.

### seed.yaml
The format should be obvious, but one important property is:
```yaml
levels:
	to: 6
```

The levels to property defines the maximum number of levels that should be cached.

### mapproxy.yaml
Note that there is likely a lot of parameters defined in this file that are unnecessary. This file is a modified example configuration that comes with the applicaiton. For the most part, modifying the properties should be fairly obvious, but there are a few key parameters that needed to be changed to make the seeding useful. The main one can be found under caches -> osm_cache:

```yaml
cache:
	type: file
	directory_layout: tms
```

This overrides the default caching behavior that outputs the file in a directory structure that isn't immediately usable. With type: file and directory_layout: tms, the output follows the /z/x/y directory structure.

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

### Helpful commands
To count all files in the tile dir:
```shell
find outputDir/ -follow -type f | wc -l
```

To count all non-empty tiles:
```shell
find outputDir/ -follow -type f -size 693c | wc -l
```
Note that in this case, empty tiles are 693 bytes - YMMV.

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
# Serving Tiles from AWS
There are several options for serving the tiles from AWS.

## Serving from CloudFront
CloudFront (CF) is the AWS CDN and is the preferred way for serving static content. There are several good reasons to use CF.

1. Some informal testing with POSTman shows that a request for a tile from S3 completes in about 100-300ms, while a request for the same tile from CF responds in about 50-80ms.

2. With CF, custom error behaviors can be defined. In the case of tiles, there may be extents that are requested by the client that don't exist. Instead of responding with a 403 like S3 does, CloudFront can be configured to handle 404s and 403s with a 200 and a specified response. So, if an image that doesn't exist is requested, CF can respond by providing a single transparent PNG. This is especially critical for sparse data sets where the majority of tiles may in fact be blank. In the above example, around 66% of files were empty. A massive amount of storage could be saved by discarding all empty files and providing instead one transparent tile with the custom error response.

To configure this custom error behavior:

1. Select the Distribution
2. Select Error Pages
3. Create Custom Error Response for 403 and 404. Set the response code to 200 and the response page path to the blank tile in the store.

## Serving from S3
Tiles can be served directly from S3. This is a little simpler, but there are a few issues with doing this. First off, S3 is not optimized for serving content, so file response latencies are likely greater than if they are served from CF. Second, there are several limitations with handling non-existant files. Unless there are tiles for the full extent of the layer as configured in the viewer, it is possible for the client to make many requests to files that don't exist. S3 will respond with a 403 for these files. There is no concept of symlinks in S3 and the best alternative option is to provide an object for each possible file with a redirect header for it. This is clearly sub-optimal. 

If performance isn't a concern and there are tiles for all possible requested extents, S3 may be fine. Several things must be configured before S3 contents can become public.

### Making bucket public
1. Go to S3 Management Console
2. Select the bucket -> Properties
3. Permissions -> Add (or Edit) bucket policy
4. Set some variant of the following policy:

```json
{
  "Version":"2012-10-17",
  "Statement":[{
    "Sid":"AllowPublicRead",
        "Effect":"Allow",
      "Principal": {
            "AWS": "*"
         },
      "Action":["s3:GetObject"],
      "Resource":["arn:aws:s3:::changeThisToYourBucketName/*"
      ]
    }
  ]
}
```

### CORS configuration
Even if the bucket contents are public, any machines attempting to access the files are likely to run into permission issues with cross-origin resources. To resolve:

1. Go to S3 Management Console
2. Select the bucket -> Properties
3. Permissions -> Add (or Edit) CORS Configuration
4. Set some variant of the following configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>Authorization</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```

### DNS mapping
See [this link](https://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html) for mapping S3 to a custom domain name.

# Cost to store in S3 and serve with CloudFront
Note that the following estimates are based on ealry 2015 prices. These regularly drop in price. For example, in late 2014 when I was first investigating this, the cost to transfer data between S3 and CF was $0.02/GB, now it is free. Edge locations were selected only for US and Europe.

Using the CONUS-scale 231m resolution example raster, with 1-12 zoom levels, the tile collection consists of about 160k tiles for 2.3GB of storage.

## One-time costs
Note that the following is based on my best estimate on how CF caches files from S3. This would assume that all files would be cached in CF one time.

| Step                                | Count and rate      | Cost  |
|-------------------------------------|---------------------|------:|
| HTTP requests to upload to S3       | 160k @ $0.005/1k    | $0.80 |
| S3 HTTP requests to cache all in CF | 160k @ $0.004/10k   | $0.07 |
| CF HTTP requests to S3              | 160k @ $0.0075/10k  | $0.12 |
|                                     |                     |       |
| Total one-time costs                | Total               | $0.99 |

So, even if this estimate is off by an order of magnitude, the one time cost is fairly trivial.

## Recurring costs
| Step                          | Count and rate         | Cost       |
|-------------------------------|------------------------|-----------:|
| Storing tiles in S3           | 2.3GB @ $0.03/GB/mo    | $0.07/mo   |
| GET requests                  | Variable @ $0.0075/10k | Variable   |
| Data transfer out to internet | Variable @ $0.085/GB   | Variable   |

Note that the data transfer cost is actually cheaper from CF than from S3. Depending on how transfer-heavy the requests are, it may end up being much more economical to go with CF than transferring data from S3, despite the slightly higher HTTP verb cost and the per-request cost to cache files.
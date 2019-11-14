# Hosting basemap.at vector tiles offline
Austria's government provides a lot of open data on its [data.gv.at- Open Data Österreich](https://www.data.gv.at) portal. One of the treasures that can be found there are offline vector tiles for [basemap.at Verwaltungsgrundkarte Österreich](https://www.data.gv.at/katalog/dataset/b694010f-992a-4d8e-b4ab-b20d0f037ff0). 

This readme will guide you through all steps required in order to host __basemap.at__ vector tiles on your private server.

Since the tiles provided are contained in an ESRI Vector Tile Package (VTPK) and this container format cannot be served by open source tile servers we have written [OpenVTPK](https://github.com/syncpoint/openvtpk) in order to repackage the tiles into an MBTiles container.

Please read our detailed instructions on how to use [OpenVTPK](https://github.com/syncpoint/openvtpk). After you have repackaged the vector tiles continue here.

## Prerequisits

We will assume that you followed the instructions provided by [OpenVTPK](https://github.com/syncpoint/openvtpk) and have your expanded basemap.at VTPK folder ready to use.

Since Docker is our best friend we are not going into a complicated setup procedure. We will use Klokan Technologies [TileServer GL](https://tileserver.readthedocs.io/en/latest/) to serve the tiles and the corresponding styles. 

## Styles included

The provided VTPK container also includes all resources required to style the vector tiles:
* fonts
* sprites
* styles 

All the resources required are located in the ```p12/resources``` subfolder of your expanded VTPK container (see description above).

Let's create a folder that will become the root folder for _TileServer GL_ (i.e. ```basemap```) and copy the folders ```fonts```, ```styles``` and ```sprites``` from the ```p12/resources``` folder. We do not need the ```infos``` folder. 

Create an additional ```tiles``` folder and put the _mbtiles_ file you want to serve here. Please follow the instructions above in order to repackage the _basemap.at_ VTPK container which we will use in this example.

After that your folder structure should look like this:

```
basemap
  fonts
    Arial Regular
    Arial Bold
    Corbel Regular
    Corbel Bold
    Corbel Italic
    Corbel Bold Italic
    Tahoma Regular
  sprites
    sprite.png
      sprite.json
      sprite@2x.png
      sprite@2x.json
  styles
    root.json
  tiles
    Basemap_20190617.mbtiles
```

## Tweaks for the basemap.at style
Unfortunately the _root.json_ style shipped with the current version of _basemap.at_ (as of October 28th 2019) contains a lot of errors:

```bash
mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'duplicate layer id STRASSENNETZ/BRUECKE/GIP_OUTSIDE_BAUWERK_L_BRÜCKE/112111/1' }
mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'duplicate layer id STRASSENNETZ/BRUECKE/GIP_OUTSIDE_BAUWERK_L_BRÜCKE/112111/0' }
mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'function value must specify stops' }
mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'function value must specify stops' }
mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'function value must specify stops' }
mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'function value must specify stops' }

(... many more lines ...) 
```

Please find a [corrected version of the style file](styles/root.json) in our repository. The changes reflect
* renaming a duplicate style layer ```STRASSENNETZ/BRUECKE/GIP_OUTSIDE_BAUWERK_L_BRÜCKE/112111/0```
* renaming a duplicate layer ```STRASSENNETZ/BRUECKE/GIP_OUTSIDE_BAUWERK_L_BRÜCKE/112111/1```
* fixing 
  ```
  mbgl: { class: 'ParseStyle',
  severity: 'WARNING',
  text: 'function value must specify stops' }
  ```
  by adding
  ```
  "stops": [
        [{"zoom": 17, "value": 0}, 0],
        [{"zoom": 18, "value": 0}, 0]
    ]
  ```



This file also contains modifications required to meet all path values we will use for the tile server:

```JSON
    "sprite": "/sprite",
    "glyphs": "/{fontstack}/{range}.pbf",
    "sources": {
        "esri": {
            "type": "vector",
            "url": "/data/basemap.at.json"
        }
    }
```

## TileServer

Now we are ready to create a configuration file (_config.js_) for _TileServer GL_:

```JSON
{
  "options": {
    "paths": {
      "root": "",
      "fonts": "fonts",
      "sprites": "sprites",
      "styles": "styles",
      "mbtiles": "tiles"
    }
  },
  "styles": {
    "basemap": {
      "style": "root.json",
      "serve_data": true
    }
  },
  "data": {
    "basemap.at": {
      "mbtiles": "Basemap_20190617.mbtiles"
    }
  },
  "settings": {
    "serve": {
      "vector": true,
      "raster": true,
      "services": true,
      "static": true
    },
    "raster": {
      "format": "PNG_256",
      "hidpi": 2,
      "maxsize": 2048
    }
  }
}
```




Open your console, change to the _basemap_ folder we created and start _TileServer GL:_

```bash
docker run --rm -it -v $(pwd):/data -p 8080:80 klokantech/tileserver-gl
Starting Xvfb on display 99
xdpyinfo:  unable to open display ":99".
xdpyinfo:  unable to open display ":99".

Starting tileserver-gl v2.6.0
Using specified config file from config.json
Starting server
Listening at http://[::]:80/
Startup complete
```

Open your favorite browser and navigate to __http://localhost:8080__ (we have mapped TCP port 80 from within the docker container to port 8080 on our local machine):

![TileServer GL Dashboard](images/tileserver-gl.png)

Click on the __Vector__ link on the dashboard in order to view your styled vector tiles. Depending on the correctness of the metadata in the VTPK file and the style rules (especially ```minzoom``` and ```maxzoom```) your viewpoint my be out of scope and you will only see a white page. Please change the zoom level and the coordinates in the browser url accordingly.

The result is beautiful and available offline:

![styled basemap](images/styled-basemap.jpg)

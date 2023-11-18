- 👋 Hi, I’m @Rakeshx609
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
Rakeshx609/Rakeshx609 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
#!/usr/bin/env bash
LOGDIR=/home/superkuh/bob/radio	# rtl_power output
OUTPUTDIR=/home/superkuh/bob/radio/www # temp dir
WEBDIR=/home/superkuh/limbo/www/radio/	# put the finished dir here
PANOJSLIB=/home/superkuh/bob/radio/panojs # contains folders: extjs,images,panojs,styles 

# Requirements: rtl_power, heatmap.py, PanoJS, imgcnv, ImageMagick
#	You have to manually edit and set the directory variables up above this.
# 	A copy of heatmap.py should be in $LOGDIR.

# imgcnv of BioImageConvert is required for slicing. There is a Debian repository for it at:
# https://bitbucket.org/dimin/bioimageconvert/wiki/Appending%20debian%20repositories 
# To fulfill dependencies I had to install libopenjpeg5 1.5.2 from utopic. 
# http://packages.ubuntu.com/pt/utopic/amd64/libopenjpeg5/download
# identify from ImageMagick is required to get x and y pixel sizes.

# The visualizations are from keenerd's rtl_power using FFT mode and his heatmap.py
# https://github.com/keenerd/rtl-sdr
# https://raw.githubusercontent.com/keenerd/rtl-sdr-misc/master/heatmap/heatmap.py

# cd ./rtl-sdr/build/src
# ./rtl_power -f 86M:1.1G:25k -g 34 -c 28% $LOGDIR/$(date +%Y-%m-%d_%H-%M --utc)_86-1100_25k.csv

# This script automates processing of heatmap.py's 50,000px wide png into 3600 smaller image
# tiles using imgcnv that PanoJS can load and present with javascript on a static website. 
# http://www.dimin.net/software/panojs/
#
# The assumption is that within the $LOGDIR the most recent csv file is what to process.

# ./rtlpower2web.sh

# Or you can pass the cvs filepath as an argument with,

# ./rtlpower2web.sh --file ~/somefile.csv

# The filename determines the final directory name. The number of seconds between time marks is,

# ./rtlpower2web.sh --tick 600 --file ~/somefile.csv

#
# It first calls heatmap.py to make the wide png. Then imgcnv slices that into a few sets of
# zoom levels of png tiles; about 3600 of them. The original and resulting filenames fill a 
# barebones PanoJS html template to generate an index.html. This is deposited in the $OUTPUTDIR
# with png tiles and PanoJS js libs to run it. 
# 
# That independent PanoJS implementation dir is transferred to the webserver. The whole 
# process takes about ~30 minutes for a 2.5 GB csv file on my i5-3750 w/16GB ram.

# PanoJS is great. The one change I had to make was  in ./panojs/pyramid_imgcnv.js, line 94,
#    return "" + l + "_" + x + "_" + y + ".jpg";//?"+level;    
# to
#    return "" + l + "_" + x + "_" + y + ".png";//?"+level; 
# Because PanoJS kept trying to use .jpg extension to load tiles and I didn't know how else to
# configure it. imgcnv can not output jpg in this mode. 
#


cd $LOGDIR

args=("$@")
COUNTER=1
while test $# -gt 0
do
    case "$1" in
        --tick) TIMETICKS=${args[$COUNTER]}
            ;;
        --file) LOGFILE=${args[$COUNTER]};FILENAME="${LOGFILE%.*}"
            ;;
    esac
    ((COUNTER++))
    shift
done
if ! [ -n "$TIMETICKS" ] ; then
    TIMETICKS=3600;
fi

if ! [ -n "$LOGFILE" ] ; then
    LOGFILE=$(ls -Art | grep ".csv" | tail -n 1) 
#    FILENAME=$(ls -Art | grep -v run | grep ".csv" | tail -n 1)
    FILENAME="${LOGFILE%.*}"
#    FILENAME="${FILENAME%.*}"
fi

#FILEDATE=$(date +%Y-%m-%d_%H-%M --utc --reference=$(ls -Art | tail -n 1))
#FILEDATE=$(date +%Y-%m-%d_%H-%M --utc --reference=$LOGFILE)


#echo "Copying logfile to temp.";
#time cp $LOGFILE run.csv

echo "Running heatmap.py on [$LOGFILE]"

#time ./heatmap.py run.csv spectrum_$FILEDATE.png
time ./heatmap.py --ytick $TIMETICKS $LOGFILE spectrum_$FILENAME.png

#DIRECTORY="$OUTPUTDIR/$FILEDATE"
DIRECTORY="$OUTPUTDIR/$FILENAME"
echo "Checking if output [$DIRECTORY] exists already..."
if [[ -d "${DIRECTORY}" && ! -L "${DIRECTORY}" ]] ; then
    echo "Dir already exists. Deleting old [$DIRECTORY]."
    rm -rf $DIRECTORY 
else
    echo "Output dir doesn't exist. Creating it."
fi
mkdir $DIRECTORY

PNGIMAGE=$(ls -Art | grep ".png" | tail -n 1)

WIDTH=$(identify -format "%w" $PNGIMAGE)
HEIGHT=$(identify -format "%h" $PNGIMAGE)

echo "Slicing images..."

time imgcnv -i $PNGIMAGE -o $DIRECTORY/spectrum.png -t png -tile 256
# -tile 256 argument defines the size of the tiles in pixels tiles will be created based on 
# the outrput file name with inserted L, X, Y, where L - is a resolution level, L=0 is native
# resolution, L=1 is 2x smaller, and so on X and Y - are tile indices in X and Y, where the
# first tile is 0,0, second in X is: 1,0 and so on ex: '-o my_file.jpg' will produce files:
# 'my_file_LLL_XXX_YYY.png' 

echo "Done, [$DIRECTORY] full of tiles."

echo "Copying PanoJS stuff..."
cp -R $PANOJSLIB/* $DIRECTORY/

echo "Generating HTML index ... [$DIRECTORY/index.html]"

cat >$DIRECTORY/index.html << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html>
  <head>
    <title>radio spectrogram</title>
    <meta http-equiv="imagetoolbar" content="no" />
    
    <style type="text/css">
      @import url(styles/panojs.css);
    </style>

    <script type="text/javascript" src="extjs/ext-core.js"></script>    
    <script type="text/javascript" src="panojs/utils.js"></script>    
    <script type="text/javascript" src="panojs/PanoJS.js"></script>
    <script type="text/javascript" src="panojs/controls.js"></script>
    <script type="text/javascript" src="panojs/pyramid_imgcnv.js"></script>
    <script type="text/javascript" src="panojs/control_thumbnail.js"></script>
    <script type="text/javascript" src="panojs/control_info.js"></script>
    <script type="text/javascript" src="panojs/control_svg.js"></script>

<style type="text/css">

body {
  font-family: sans-serif;
  margin: 0;
  padding: 10px;
  color: #000000;
  background-color: #FFFFFF;
  font-size: 0.7em;
}

</style> 
                
<script type="text/javascript">
// <![CDATA[

PanoJS.MSG_BEYOND_MIN_ZOOM = 0;
PanoJS.MSG_BEYOND_MAX_ZOOM = 8;
var viewer1 = null;

function createViewer( viewer, dom_id, url, prefix, w, h ) {
    if (viewer) return;
  
    var MY_URL      = url;
    var MY_PREFIX   = prefix;
    var MY_TILESIZE = 256;
    var MY_WIDTH    = w;
    var MY_HEIGHT   = h;
    var myPyramid = new ImgcnvPyramid( MY_WIDTH, MY_HEIGHT, MY_TILESIZE);
    
    var myProvider = new PanoJS.TileUrlProvider('','','');
    myProvider.assembleUrl = function(xIndex, yIndex, zoom) {
        return MY_URL + '/' + MY_PREFIX + myPyramid.tile_filename( zoom, xIndex, yIndex );
    }    
    
    viewer = new PanoJS(dom_id, {
        tileUrlProvider : myProvider,
        tileSize        : myPyramid.tilesize,
        maxZoom         : myPyramid.getMaxLevel(),
        imageWidth      : myPyramid.width,
        imageHeight     : myPyramid.height,
        blankTile       : 'images/blank.gif',
        loadingTile     : 'images/progress.gif'
    });

    Ext.EventManager.addListener( window, 'resize', callback(viewer, viewer.resize) );
    viewer.init();
};


function initViewers() {
 // viewer, dom_id, url, prefix, w, h //
  createViewer( viewer1, 'viewer1', '/radio/$FILENAME', 'spectrum_', $WIDTH,  $HEIGHT );
}
  
Ext.onReady(initViewers);

// ]]>
</script>

</head>
<body>
    
  <h1>rtlsdr spectrogram - $FILENAME</h1>

  <div style="width: 100%; height: 900px;"> 
      <div id="viewer1" class="viewer" style="width: 100%; height: 100%;" ></div>
  </div>
    
</body>
</html>
EOF

echo "Copying everything to [$WEBDIR] ..."
time cp -R $DIRECTORY/ $WEBDIR/
# time parallel -j10 cp {} $WEBDIR/ ::: $DIRECTORY/*
# cd $DIRECTORY; time tar cf - $FILENAME/ | time ssh username@server.com 'cd /home/user/www/radio && tar xf -'


#echo "Deleting tmp image [$PNGIMAGE]: Disabled."
#rm $PNGIMAGE
echo "Done."

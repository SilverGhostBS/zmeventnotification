<!-- TOC -->

- [Note](#note)
- [Limitations](#limitations)
- [What](#what)
- [Installation](#installation)
    - [Option 1: Automatic install](#option-1-automatic-install)
    - [Option 2: Manual install](#option-2-manual-install)
- [Post install steps](#post-install-steps)
- [Test operation](#test-operation)
- [Troubleshooting](#troubleshooting)
- [Types of detection](#types-of-detection)
    - [RECOMMENDED: detect_yolo.py:  using OpenCV DNN with YoloV3 (much slower, accurate)](#recommended-detect_yolopy--using-opencv-dnn-with-yolov3-much-slower-accurate)
    - [detect_yolo.py:  using OpenCV DNN with Tiny YoloV3 (almost comparable with HOG in speed, more accurate)](#detect_yolopy--using-opencv-dnn-with-tiny-yolov3-almost-comparable-with-hog-in-speed-more-accurate)
    - [detect_hog.py: using OpenCV SVM HOG (very fast, not accurate)](#detect_hogpy-using-opencv-svm-hog-very-fast-not-accurate)
- [Performance comparison](#performance-comparison)

<!-- /TOC -->


### Note
**Please don't ask me basic questions like "pip command not found" or "cv2 not found" - what do I do?**
**Hooks require some terminal knowledge and familiarity with troubleshooting**
**I don't plan to provide support for these hooks. They are for reference only**

### Limitations

* Only tested with ZM 1.32+. May or may not work with older versions

### What
This is an example of how you can use the `hook` feature of the notification server
to invoke a custom script on the event before it generates an alarm. This implements a hook script that detects
objects using Machine Learning for events. If it matches the objects you are interested in, it will send a notification.

There are two sample detection scripts. You can switch between them by changing the value of
`DETECTION_SCRIPT` in `detect_wrapper.sh`

**Both of these scripts require setup, please run from command line first and 
make sure deps are installed**

Please don't ask me questions on how to use them. Please read the comments and figure it out.

Try to keep the images less than 800px on the largest side. The larger the image, the longer
it will take to detect

### Installation




#### Option 1: Automatic install

*  Only tested with Python2
*  You need to have `pip` installed. On ubuntu, it is `sudo apt install python-pip`, or see [this](https://pip.pypa.io/en/stable/installing/)
*  Clone the event server and go to the `hook` directory 


```bash
git clone https://github.com/pliablepixels/zmeventnotification # if you don't already have it downloaded

cd zmeventnotification
```

* (OPTIONAL) Edit `hook/detect_wrapper.sh` and change:
    * `CONFIG_FILE` to point to the right config file, if you changed paths
    * `DETECTION_SCRIPT` if you want to change from YOLO to HOG

```
sudo ./install.sh # and follow the prompts
```

#### Option 2: Manual install 

If automatic install fails for you, or you like to be in control:


```bash
git clone https://github.com/pliablepixels/zmeventnotification # if you don't already have it downloaded
cd zmeventnotification/hooks
```

* Install the object detection dependencies:
```bash
sudo pip install -r  requirements.txt 
```

* You now need to download configuration and weight files that are required by the machine learning magic. Note that you don't have to put them in `/var/lib/zmeventnotification` -> use whatever you want (and change variables in `detect_wrapper.sh` script if you do) 

```bash
sudo mkdir -p /var/lib/zmeventnotification/images
sudo mkdir -p /var/lib/zmeventnotification/models
sudo mkdir -p /etc/zm

# if you want to use YoloV3 (slower, accurate)
sudo mkdir -p /var/lib/zmeventnotification/models/yolov3 # if you are using YoloV3
sudo wget https://raw.githubusercontent.com/pjreddie/darknet/master/cfg/yolov3.cfg -O /var/lib/zmeventnotification/models/yolov3/yolov3.cfg
sudo wget https://raw.githubusercontent.com/pjreddie/darknet/master/data/coco.names -O /var/lib/zmeventnotification/models/yolov3/yolov3_classes.txt
sudo wget https://pjreddie.com/media/files/yolov3.weights -O /var/lib/zmeventnotification/models/yolov3/yolov3.weights

--OR--

# if you want to use TinyYoloV3 (faster, less accurate)
sudo mkdir -p /var/lib/zmeventnotification/models/tinyyolo # if you are using TinyYoloV3
sudo wget https://pjreddie.com/media/files/yolov3-tiny.weights -O /var/lib/zmeventnotification/models/tinyyolo/yolov3-tiny.weights
sudo wget https://raw.githubusercontent.com/pjreddie/darknet/master/cfg/yolov3-tiny.cfg -O /var/lib/zmeventnotification/models/tinyyolo/yolov3-tiny.cfg
sudo wget https://raw.githubusercontent.com/pjreddie/darknet/master/data/coco.names -O /var/lib/zmeventnotification/models/tinyyolo/yolov3-tiny.txt
```

* Copy over the object detection config file

```bash
sudo cp objectconfig.ini /etc/zm
```


* Now make sure it all RW accessible by `www-data` (or `apache`)
```
sudo chown -R www-data:www-data /var/lib/zmeventnotification/ #(change www-data to apache for CentOS/Fedora)
```

* (OPTIONAL) Edit `detect_wrapper.sh` and change:
    * `CONFIG_FILE` to point to the right config file, if you changed paths
    * `DETECTION_SCRIPT` if you want to change from YOLO to HOG


* Now copy your detection files to `/usr/bin` 
```
sudo cp detect_* /usr/bin
```


### Post install steps

* Make sure you edit your installed `objectconfig.ini` to the right settings. You MUST change the `[general]` section for your own portal.
* Make sure the `CONFIG_FILE` variable in `detect_wrapper.sh` is correct 


### Test operation
```
sudo -u www-data /usr/bin/detect_wrapper.sh <eid> <mid> # replace www-data with apache if needed
```

This will try and download the configured frame for alarm <eid> and analyze it. Replace with your own EID (Example 123456)
The files will be in `/var/lib/zmeventnotification/images`
For example: 
if you configured `frame_id` to be `bestmatch` you'll see two files `<eid>-alarm.jpg` and `<eid>-snapshot.jpg`
If you configured `frame_id` to be `snapshot` or a specific number, you'll see one file `<eid>.jpg`

The `<mid>` is optional and is the monitor ID. If you do specify it, it will pick up the right mask to apply (if it is in your config)

The above command will also try and run detection.

If it doesn't work, go back and figure out where you have a problem

* Other configuration notes, after you get everything working
    * Set `delete_after_analyze` to `yes` so that downloaded images are removed after analysis. In the default installation, the images are kept in `/var/lib/zmeventnotification/images` so you can debug.
    * Remember these rules:
        * `frame_id=snapshot` will work for any ZM >= 1.32
        * If you are running ZM < 1.33, to enable `bestmatch` or `alarm` you need to enable the monitor to store JPEG frames in its ZM monitor->storage configuration in ZM 
        * If you are running ZM >= 1.33, you can use all fid modes without requiring to enable frames in storage


### Troubleshooting

* In general, I expect you to debug properly. Please don't ask me basic questions without investigating logs yourself
* Always run `detect_wrapper.sh` in manual mode first to make sure it works
* Set `log_level` to `debug` in `objdetect.ini` before you run it manually
* When you run in manually, view system logs in another window to look for errors (`tail -f /var/log/syslog | grep yolo`)
* One of the big reasons why object detection fails is because the hook is not able to download the image to check. This may be because your ZM version is old or other errors. Some common issues:
    * Make sure your `objectconfig.ini` section for `[general]` are correct (portal, user,admin)
    * For object detection to work, the hooks expect to download images of events using `https://yourportal/zm/?view=image&eid=<eid>&fid=snapshot` and possibly `https://yourportal/zm/?view=image&eid=<eid>&fid=alarm`
    * Open up a browser, log into ZM. Open a new tab and type in `https://yourportal/zm/?view=image&eid=<eid>&fid=snapshot` in your browser. Replace `eid` with an actual event id. Do you see an image? If not, you'll have to fix/update ZM. Please don't ask me how. Please post in the ZM forums
    * Open up a browser, log into ZM. Open a new tab and type in `https://yourportal/zm/?view=image&eid=<eid>&fid=alarm` in your browser. Replace `eid` with an actual event id. Do you see an image? If not, you'll have to fix/update ZM. Please don't ask me how. Please post in the ZM forums
    



### Types of detection

#### RECOMMENDED: detect_yolo.py:  using OpenCV DNN with YoloV3 (much slower, accurate)

The detection uses OpenCV's DNN module and YoloV3 to predict multiple labels with score.

You can manually invoke it to test:

```bash
./sudo -u www-data /usr/bin/detect_yolo.py --config /etc/zm/objectconfig.ini  --eventid <eid> --monitorid <mid>
```
The `--monitorid <mid>` is optional and is the monitor ID. If you do specify it, it will pick up the right mask to apply (if it is in your config)


If you are using YOLO models, you will need the following data files (if you followed the installation directions, you already have them):
* weights: https://pjreddie.com/media/files/yolov3.weights
* config: https://raw.githubusercontent.com/pjreddie/darknet/master/cfg/yolov3.cfg
* labels: https://raw.githubusercontent.com/pjreddie/darknet/master/data/coco.names



#### detect_yolo.py:  using OpenCV DNN with Tiny YoloV3 (almost comparable with HOG in speed, more accurate)

The detection uses OpenCV's DNN module and Tiny YoloV3 to predict multiple labels with score.

You can manually invoke it to test:

```bash
./sudo -u www-data /usr/bin/detect_yolo.py --config /etc/zm/objectconfig.ini  --eventid <eid> --monitorid <mid>
```
The `--monitorid <mid>` is optional and is the monitor ID. If you do specify it, it will pick up the right mask to apply (if it is in your config)


If you are using YOLO models, you will need the following data files (if you followed the installation directions, you already have them):
* weights: https://pjreddie.com/media/files/yolov3-tiny.weights
* config:  https://raw.githubusercontent.com/pjreddie/darknet/master/cfg/yolov3-tiny.cfg
* labels:  https://raw.githubusercontent.com/pjreddie/darknet/master/data/coco.names

(Note: `coco.names` is the label file the script needs. It is common for both tiny or regular yolo)

#### detect_hog.py: using OpenCV SVM HOG (very fast, not accurate)

You can manually invoke it to test:

```bash
./sudo -u www-data /usr/bin/detect_hog.py --config /etc/zm/objectconfig.ini  --eventid <eid> --monitorid <mid>
```
The `--monitorid <mid>` is optional and is the monitor ID. If you do specify it, it will pick up the right mask to apply (if it is in your config)

The detection uses a very fast, but not very accurate OpenCV model (hog.detectMultiScale). 
The good part is that it is extremely fast can can be used for realtime needs. 
Fiddle with the config settings in the `[hog]` section (stride/scale) to get more accuracy at the cost of speed.


### Performance comparison

DNNs perform very well on a GPU. My ZM server doesn't have a GPU. 
On a Intel Xeon 3.16GHz 4Core machine:
- HOG takes 0.24s
- YOLOv3 with tiny-yolo takes 0.32s
- YOLOv3 takes 2.4s


As always, if you are trying to figure out how this works, do this in 3 steps:

**STEP 1: Make sure the scripts(s) work**
- Run the python script manually to see if it works (refer to sections above on how to run them manuall)
- `./detect_wrapper.sh <eid> <mid>` --> make sure it downloads a proper image for that eid. Make sure it correctly invokes detect_xxx.py If not, fix it. (`<mid>` is optional and is used to apply a crop mask if specified)
- Make sure the `image_path` you've chosen in the config file is WRITABLE by www-data (or apache) before you move to step 2

**STEP 2: run zmeventnotification in MANUAL mode**
* `sudo zmdc.pl stop zmeventnotification.pl`
*  change verbose to 1 in `zmeventnotification.ini`
*  `sudo -u www-data ./zmeventnotification.pl  --config ./zmeventnotification.ini`
*  Force an alarm, look at logs

**STEP 3: integrate with the actual daemon**

* You should know how to do this already

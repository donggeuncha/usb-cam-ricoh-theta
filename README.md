usb_cam_ricoh_theta_ros
===============

# How to set up RICOH THETA for usb_cam ROS package
This tutorial is based on [**RICOH THETA Development on Linux**](https://codetricity.github.io/theta-linux/software/)

## Cloning the repository
There are two ways to clone this repository.

* Clone the main repository and submodules at the same time.
```
$ cd ~/catkin_ws/src/
$ git clone --recursive https://github.com/donggeuncha/usb-cam-ricoh-theta.git
```

* Update submodules after cloning the main repository. (You don't have to do this if you have cloned the repository with '--recursive' option)
```
$ cd ~/catkin_ws/src/
$ git clone https://github.com/donggeuncha/usb-cam-ricoh-theta.git
$ cd usb-cam-ricoh-theta
$ git submodule update --remote --merge
```

## Getting Stream on /dev/video0

### Build and install v4l2loopback
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd v4l2loopback
$ make 
$ sudo make install
$ sudo depmod -a
```

After installation connect your device via USB, turn it on, select LIVE mode and run:
```
$ sudo modprobe v4l2loopback
$ ls /dev | grep video
video0
video1
video2
```

### Install required dependencies
```
$ sudo apt install libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-doc gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio libgstreamer-plugins-base1.0-dev libjpeg-dev
```

### Verify nvdec installed
If `nvdec` is not installed:
```
$ gst-inspect-1.0 nvdec
No such element or plugin 'nvdec'
```
or if `nvdec` is installed:
```
$ gst-inspect-1.0 | grep nvdec
nvdec:  nvdec: NVDEC video decoder
```

### Git checkout gst-plugins-bad
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd gst-plugins-bad

# verify gstreamer version
$ gst-inspect-1.0 --version
gst-inspect-1.0 version 1.14.5
GStreamer 1.14.5
https://launchpad.net/distros/ubuntu/+source/gstreamer1.0

$ git checkout {your gstreamer version}
Note: checking out '1.14.5'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at dc1b94a15 Release 1.14.5

# verify that you're on the correct branch
$ git branch
* (HEAD detached at 1.14.5)
  master
```

### Download NVIDIA CODEC SDK and copy files
1. Download [NVIDIA CODEC SDK](https://developer.nvidia.com/nvidia-video-codec-sdk/download).
2. Unzip and copy files.
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd gst-plugins-bad
$ cp /usr/local/cuda/include/cuda.h sys/nvenc
$ cp {path to NVIDIA CODEC SDK}/Interface/nvEncodeAPI.h sys/nvenc
$ cp {path to NVIDIA CODEC SDK}/Interface/cuviddec.h sys/nvdec
$ cp {path to NVIDIA CODEC SDK}/Interface/nvcuvid.h sys/nvdec
```

### Build and install NVIDIA CODEC plug-in
```
$ NVENCODE_CFLAGS="-I${HOME}/catkin_ws/src/usb-cam-ricoh-theta/gst-plugins-bad/sys/nvenc" ./autogen.sh --disable-gtk-doc --with-cuda-prefix="/usr/local/cuda"

$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd gst-plugins-bad
$ cd sys/nvenc
$ make
$ sudo cp .libs/libgstnvenc.so /usr/lib/x86_64-linux-gnu/gstreamer-1.0/
$ cd ../nvdec
$ make
$ sudo cp .libs/libgstnvdec.so /usr/lib/x86_64-linux-gnu/gstreamer-1.0/
```

### Verify NVIDIA CODEC plug-in installed
```
$ gst-inspect-1.0 | grep nvenc
nvenc:  nvh264enc: NVENC H.264 Video Encoder
$ gst-inspect-1.0 | grep nvdec
nvdec:  nvdec: NVDEC video decoder
```

### Build and install libuvc-theta
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd libuvc-theta
$ mkdir build
$ cd build
$ cmake ..
$ make
$ sudo make install
```

### Build libuvc-theta-sample
This repository contains customed `libuvc-theta-sample` submodule (forked from [ricohapi/libuvc-theta-sample](https://github.com/ricohapi/libuvc-theta-sample)) with modifications to add support nvdec hardware decoding plug-in for NVIDIA GPUs.

Before compiling you have to replace `/dev/video0` with your device interface number. (You may have to try each number to find your device.)
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd libuvc-theta-sample/gst
$ make
```

### Load and test v4l2loopback module
This assumes that you have replaced the device interface number in `gst_viewer.c`.
Before test connect your device via USB, turn it on, select LIVE mode.
```
$ sudo modprobe v4l2loopback
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd libuvc-theta-sample/gst
$ ./gst_loopback nvdec /dev/video0
pipe_proc = nvdec ! gldownload ! videoconvert n-thread=0 ! video/x-raw,format=I420 ! identity drop-allocation=true ! v4l2sink device=/dev/video0 qos=false sync=false
start, hit any key to stop
$ cvlc v4l2:///dev/video0
VLC media player 3.0.8 Vetinari (revision 3.0.8-0-gf350b6b5a7)
[000055d39f03d240] dummy interface: using the dummy interface module...
```
If you execute `gst_loopback` with no arguments, `gst_loopback` runs without `nvdec`.

### Build usb_cam ROS package
This repository contains customed `usb_cam` submodule (forked from [ros-drivers/usb_cam](https://github.com/ros-drivers/usb_cam)) with modifications to add support for `YUV420 (yu12)` pixel format.
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd usb_cam
$ git checkout feature/YUV420
Switched to branch 'feature/YUV420'
Your branch is up to date with 'origin/feature/YUV420'.
$ git branch
$ cd ~/catkin_ws
$ catkin_make -DCMAKE_BUILD_TYPE=Release
```

### Run v4l2loopback and usb_cam ROS node
Connect your device via USB, turn it on, select LIVE mode.
Run `gst_loopback` in `nvdec` mode.
```
$ cd ~/catkin_ws/src/usb-cam-ricoh-theta
$ cd libuvc-theta-sample/gst
$ ./gst_loopback nvdec /dev/video0
pipe_proc = nvdec ! gldownload ! videoconvert n-thread=0 ! video/x-raw,format=I420 ! identity drop-allocation=true ! v4l2sink device=/dev/video0 qos=false sync=false
start, hit any key to stop
```
Run `usb_cam` node with following ROS launch file in other terminal.
```
<launch>
  <node name="usb_cam" pkg="usb_cam" type="usb_cam_node" output="screen" >
    <param name="video_device" value="/dev/video0" />
    <param name="image_width" value="3840" />
    <param name="image_height" value="1920" />
    <param name="pixel_format" value="yu12" />
    <param name="camera_frame_id" value="camera" />
    <param name="io_method" value="mmap"/>
  </node>
</launch>
```
Run ROS `rviz` and add `/usb_cam/image_raw` topic to view current v4l2loopback stream.

# How to load v4l2loopback automatically on boot
Add following lines to `/etc/modules-load.d/modules.conf`.
```
# for RICOH THETA live streaming
# v4l2loopback device on /dev/video0. specify in gst_viewer.c
v4l2loopback
```

## Check kernel module load
```
$ lsmod | grep v4l2loopback
v4l2loopback           45056  0
```

# How to get video device output
```
v4l2-ctl --list-formats-ext --device  /dev/video0
ioctl: VIDIOC_ENUM_FMT
	Index       : 0
	Type        : Video Capture
	Pixel Format: 'YU12'
	Name        : Planar YUV 4:2:0
		Size: Discrete 3840x1920
			Interval: Discrete 0.033s (30.000 fps)
```

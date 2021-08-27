# nginx-hls-server

## Objective

To create a video stream and serve as a HLS endpoint via NGINX.

## Prerequisites

The steps listed here should work for most *nix devices, as long as `ffmpeg` & `nginx` are supported & installed.

### ffmpeg

Debian-based:
```sh
sudo apt-get install ffmpeg
```

Arch-based:
```sh
sudo pacman -S ffmpeg
```

RHEL:
```sh
yum install ffmpeg
```

MacOS:
```sh
brew install ffmpeg
```

### NGINX

As we need to use the RTMP Module in NGINX, which might not be present in certain build, we will clone the git repo and compile `nginx`:
```sh
cd /path/to/build/dir
git clone https://github.com/arut/nginx-rtmp-module.git
git clone https://github.com/nginx/nginx.git
cd nginx
./auto/configure --add-module=../nginx-rtmp-module
make
sudo make install
```

This will install NGINX in `/usr/local/nginx`.

## Setup

### 1. Create ffmepg HLS stream

Here is an example to set up a HLS stream at `/tmp/hls/` with following settings:

#### Input
- Input device: `/dev/video0`
- Resoltuion: `1280x720`
- Framerate: `30`

#### HLS stream
- Encoding preset: `superfast`
- Target (average) bit rate: `5000k`
- Codec: `h264`
- Segment time interval: `6` seconds
- Playlist size: `10` segments
- Wrap limit: `40` segements
- HLS flag: `delete_segments`
- Segement deletion threshold: `1`

```sh
ffmpeg -video_size 1280x720 \
       -framerate 30 \
       -i /dev/video0 \
       -f hls \
       -preset superfast \
       -c:v libx264 \
       -b:v 5000k \
       -hls_time 6 \
       -hls_list_size 10 \
       -hls_wrap 40 \
       -hls_delete_threshold 1 \
       -hls_flags delete_segments \
       /tmp/hls/stream.m3u8
```

You can run it with `&` or other tools (e.g. `screen` or `nohup`) to make it run in background.

### (Optional) Find out more on your video input device

If you have difficulties in finding the right device/config/parameters, you can view the details device info by using `v4l2-ctl`.

To use `v4l2-ctl`, you need to have `v4l-utils` installed. Please install it using package manager of your distro.

For example:
```sh
sudo apt install v4l-utils
```

Then you can list all of the device details using the following command:
```sh
v4l2-ctl --all
```
or 
```sh
v4l2-ctl --list-devices
```

### 2. Set up NGINX 

Modify `/usr/local/nginx/conf/nginx.conf` so that it serves contents in `/tmp/hls` as HLS stream:
```nginx
worker_processes  1;
user	pi;

events {
    worker_connections  1024;
}
 
http { 
    default_type application/octet-stream;
 
    server { 
        listen 80; 
        location /live { 
            alias /tmp/hls/; 
        } 
    }
 
    types {
        application/vnd.apple.mpegurl m3u8;
        video/mp2t ts;
        text/html html;
    } 
}
```

Test the config file by running:
```sh
/usr/local/nginx/sbin/nginx -t
```

Start the nginx server:
```sh
/usr/local/nginx/sbin/nginx
```

## Test

Open VLC player and connect to `http://<ip-address>/live/stream.m3u8`.

## Optional

Add RTMP functionality to the server so that it can accept RTMP streams and process into HLS stream.
```nginx
rtmp { 
    server { 
        listen 1935; 
        application live { 
            live on; 
            interleave on;
 
            hls on; 
            hls_path /tmp/hls/; 
            hls_fragment 15s; 
        } 
    } 
} 
```

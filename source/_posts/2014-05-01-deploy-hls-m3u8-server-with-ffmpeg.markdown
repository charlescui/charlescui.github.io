---
layout: post
title: "部署基于FFMPEG的HLS直播视频服务 Deploy HLS(m3u8) Server with FFMPEG"
date: 2014-05-01 08:53:59 +0800
comments: true
categories: 项目 视频
---
这是一次搭建`直播`HLS服务器的经历，通过以下介绍，可以完成一套完整的直播HLS服务的部署。

<!--more-->

## 安装ffmpeg

下载最新代码 :

`git clone https://github.com/FFmpeg/FFmpeg.git`

安装以下依赖的库:

    sudo apt-get install libass-dev -y
    sudo apt-get install libfaac-dev -y
    sudo apt-get install libmp3lame-dev -y
    sudo apt-get install librtmp-dev -y
    sudo apt-get install libtheora-bin -y
    sudo apt-get install libtheora-dev -y
    sudo apt-get install libvorbis -y
    sudo apt-get install libvorbis-dev -y
    sudo apt-get install libvpx-dev -y
    sudo apt-get install libx264-dev -y

编译

- ./configure --enable-gpl --enable-libass --enable-libfaac --enable-libmp3lame --enable-librtmp --enable-libtheora --enable-libvorbis --enable-libvpx --enable-libx264 --enable-nonfree --enable-version3 --prefix=/usr/local/ffmpeg
- make -j8
- sudo make install

安装成功后，更新环境变量:

`export PATH=/usr/local/ffmpeg/bin:$PATH`

## 启动ffmpeg

###需求:

- 输入源是UDP的组播信号
- 输出一个m3u8文件
- 文件中每条记录的URL PATH前缀为:`/hunanweishi`
- 其中每条记录长10秒
- 每个切片文件格式为ts格式
- 每个切片文件名称格式为:`seg-%03d.ts`

启动命令如下:
`ffmpeg -i udp://xxx.xxx.xxx.xxx:30000 -f segment -codec copy -map 0 -vbsf h264_mp4toannexb -flags -global_header -segment_format mpegts -segment_list hunanweishi.m3u8 -segment_time 10 -segment_list_entry_prefix /hunanweishi/ -segment_list_size 5 seg-%03d.ts`

关键参数解释:

- ‘segment_format format’
    - Override the inner container format, by default it is guessed by the filename extension.
- ‘segment_list name’
    - Generate also a listfile named name. If not specified no listfile is generated.
- ‘segment_list_size size’
    - Update the list file so that it contains at most the last size segments. If 0 the list file will contain all the segments. Default value is 0.
- ‘segment_list_entry_prefix prefix’
    - Prepend prefix to each entry. Useful to generate absolute paths. By default no prefix is applied.
- ‘segment_time time’
    - Set segment duration to time, the value must be a duration specification. Default value is "2". See also the ‘segment_times’ option.
    - Note that splitting may not be accurate, unless you force the reference stream key-frames at the given time. See the introductory notice and the examples below.
- ‘segment_times times’
    - Specify a list of split points. times contains a list of comma separated duration specifications, in increasing order. See also the ‘segment_time’ option.

其余参数见官网：[ffmpeg的segment官网介绍](https://www.ffmpeg.org/ffmpeg-formats.html#segment_002c-stream_005fsegment_002c-ssegment)

备注:由于是在screen中启动，所以没有将其变为daemon进程。如果需要改变视频分辨率、码流，可以自行增加ffmpeg参数。

## m3u8内容分析

#### 截取hunanweishi.m3u8的内容片段

    #EXTM3U
    #EXT-X-VERSION:3
    #EXT-X-MEDIA-SEQUENCE:0
    #EXT-X-ALLOW-CACHE:YES
    #EXT-X-TARGETDURATION:11
    #EXTINF:9.983689,
    /huashufuwu/seg-000.ts
    #EXTINF:10.120000,
    /huashufuwu/seg-001.ts

#### 注意
- /huashufuwu/,这个文件前缀是ffmpeg命令中，segment_list_entry_prefix参数指定的，可以帮助使用者在配置Nginx的时候，规划静态资源路径.
- segment_list_size参数指定一个m3u8文件中有多少个视频，默认为0.
    - 如果不设置，m3u8文件中会包含所有生成的视频片段，不建议用默认参数.
    - 如果设置，比如我们设置为5，ffmpeg会在生成到第`n*5-1`个文件时候，更新m3u8文件。我们可以利用这个更新机制，实现的清理过期视频片段，否则磁盘会被用光。

## Ruby监控脚本

为了避免生成太多ts文件，没有及时删除导致占用磁盘空间，所以需要一个删除ts文件的方案。

### 方案

- 采用`inotify`内核调用机制，避免轮询导致消耗资源，这样可以一个监控进程监控多个文件，而且只占用很少的系统资源。
- 先对m3u8文件注册监听，当得到修改信号时，根据m3u8当前的sequence，查找已过期的ts文件列表，并删除。
- 但要保留部分近期的ts文件，因为网速慢的用户可能还没有下载完。

### 实现

- 生成文件:`touch clean.rb`
- 使其可执行:`chmod +x clean.rb`
- 写入下面代码

代码如下:

    #!/usr/bin/env ruby
    require "fileutils"
    require "eventmachine"

    $fnreg = /.*seg-(\d+)\.ts$/
    $m3u8 = ARGV[0]

    module Handler
        def file_modified
            puts "#{path} modified"
            clean
        end

        def file_moved
            puts "#{path} moved"
        end

        def file_deleted
            puts "#{path} deleted"
        end

        def unbind
            puts "#{path} monitoring ceased"
        end

        def clean
            seq = `cat #{path} |grep "#EXT-X-MEDIA-SEQUENCE:"|grep -v grep|cut -f2 -d\:`.chomp.to_i
            Dir[File.join(File.dirname($m3u8), '*.ts')].each{|x|
                name = File.basename(x)
                if($fnreg =~ name) and ($1.to_i < (seq - 3))
                    puts "Delete expired file #{x}, and seq #{$1}"
                    FileUtils.rm_f(x)
                end
            }
        end
    end

    Signal.trap('QUIT'){EM.stop;puts "Stop monit #{$m3u8}."}
    Signal.trap('INT'){EM.stop;puts "Stop monit #{$m3u8}."}

    # for efficient file watching, use kqueue on Mac OS X
    EventMachine.kqueue = true if EventMachine.kqueue?

    EventMachine.run {
        EventMachine.watch_file($m3u8, Handler)
    }
 
执行：`./clean.rb hunanweishi.m3u8`

Ruby的Eventmachine库支持各种异步事件，其中包括inotify。当被监控的文件产生了删除、移动、修改等操作，监控器(Handler)都会得到回掉。

监控程序的输出结果片段如下：

    Delete expired file ./seg-13602.ts, and seq 13602
    hunanweishi.m3u8 modified
    hunanweishi.m3u8 modified
    Delete expired file ./seg-13603.ts, and seq 13603
    hunanweishi.m3u8 modified
    hunanweishi.m3u8 modified
    Delete expired file ./seg-13604.ts, and seq 13604
    hunanweishi.m3u8 modified
    hunanweishi.m3u8 modified
    Delete expired file ./seg-13605.ts, and seq 13605

## Nginx配置

对于Nginx来说，HLS可以当作一个静态服务器来处理，所以配置如下:

    server {
            listen   22280; ## listen for ipv4; this line is default and implied
            root /home/www/.ffserver/live;
            index index.html index.htm;
            location / {
                    autoindex on;
                    try_files $uri $uri/ /index.html;
            }
    }

在conf/mime.types中增加如下类型:

    application/x-mpegURL                   m3u8;
    video/MP2T                              ts;

重启Nginx

## 测试

使用VLC打开m3u8地址：`http://xxx.xxx.xxx.xxx:22280/hunanweishi/hunanweishi.m3u8`，可以看到直播节目了。

### 测试结果

![直播截图](/images/posts/vlcsnap-2014-05-01-09h56m04s143.png "湖南卫视")
# MWA相干脉冲星数据处理

#### 1. 加载脉冲星处理软件
    source /usr/local/astrosoft/start_cfitsio.rc
    source /usr/local/astrosoft/start_dspsr.rc
    source /usr/local/astrosoft/start_pgplot.rc
    source /usr/local/astrosoft/start_presto.rc
    source /usr/local/astrosoft/start_psrchive.rc
    source /usr/local/astrosoft/start_tempo2.rc
    source /usr/local/astrosoft/start_tempo.rc

方便起见可以将以上命令粘帖至 `~/.bashrc` 文件的末尾，这样每次登录时会自动加载以上软件

#### 2. CRATIV服务器上已有的MWA脉冲星数据

已知脉冲星的参数par file
> /home/share/data/mwa_pulsar/pulsar_par_file

MWA相干叠加的脉冲星数据示例
>/home/share/data/mwa_pulsar/coherent_beam/1140177088/J0630-2834

#### 3. 处理命令

    dspsr -cont -U 600 -E /home/share/data/pulsar_par_file/J0630-2834.par -K -b 128 -A -L 10 -d 4 -O ~/J0630-2834_n128_ch133-156 1140177088_ch133-156_000？.fits

其中，`-cont -U 600`表示使用600Mb内存，`-E` 后面是脉冲星.par文件，`-K` 表示做消色散处理，`-b` 表示将每一个脉冲周期分成多少个time bin，`-A` 表示输出文件中包含多个时间积分段，`-L` 表示几秒钟为一个时间积分段（示例中为10s一段），`-d 4` 表示记录全部四个偏振信息，`-O` 后面跟输出文件位置。最后给出输入文件，支持通配符。运行结束后生成.ar文件。

    pam -e ar12 -T -R 44.63 J0630-2834_n128_ch133-156.ar

将psrcat中的RM信息写入.ar文件的头文件，`-e` 新生成文件的后缀，`-T`,将所有时间积分段叠加在一起，`-R` RM值。

    pav -SFT ~/J0630-2834_n128_ch133-156.ar
    pav -SFT ~/J0630-2834_n128_ch133-156.ar12
画图命令，`S` 表示画出总强度&线偏振&圆偏振，`F` 表示将所有频率段叠加，`T` 表示将所有时间积分段叠加。可对比进行RM改正和未进行RM改正的偏振轮廓。  
如需输出.ps格式图片，可在命令中加入 `-g[output].ps/cps`


    pdv -t -K -FT -Z J0630-2834_n128_ch133-156.ar > J0630-2834_n128_ch133-156.txt
输出脉冲轮廓文本文件，`-t` 表示 ascii 格式输出，`-K`表示第一行以#开始，`-F` 表示将所有频率段叠加，`-T` 表示将所有时间积分段叠加，`-Z`表示输出线偏角。


# MWA-VCS 原始数据处理
MWA-VCS原始数据在pawsey的处理流程
### 0. pawsey的服务器
服务器系统简介参考 https://support.pawsey.org.au/documentation/display/US/Supercomputing+Documentation

我们常用的主要有
1. Galaxy： 用于计算和数据处理
2. Zeus： 用于下载磁带上的存档数据

### 1. 数据下载
下载某一观测的全部数据

    process_vcs.py -m download -o [obs ID] -a

如仅需下载某一观测中一段时间的数据

    process_vcs.py -m download -o [obs ID] -b [begin (GPS seconds)] -e [end (GPS seconds)]

查看下载进度

    squeue -p copyq -M zeus -u $USER

查看某一观测实际开始和结束的时间

    file_maxmin.py [obs ID]

### 2. 生成非相干叠加数据
把下载下来的每1秒一个文件的非相干数据生成每200秒一个文件的psrfits格式文件

    create_psrfits.sh [obsID]

将全部24个coarse channel (每个频率通道带宽1.28MHz)合并成30.72MHz

    splice_psrfits [lowest channel] [second lowest channel] [...] [suffix]

### 3. Calibration
生成相干叠加数据，首先需要进行Calibration

1 下载calibrator观测

    process_vcs.py -m download_cal -o [target obs ID] -O [cal obs ID]

2 生成source list

    srclist_by_beam.py -m [metafits file] -n [N sources] -s [base RTS source list]

3 进行Calibration

    calibrate_vcs.py -o [obs ID] -O [calibrator obs ID] -m [metafits file for calibrator observation] -s [path to source list file] --gpubox_dir [path to visibilities] --rts_output_dir [path to write solutions and config files] [--nosubmit] [--offline]

### 4. 生成相干叠加文件
针对在primary beam中的某一 RA，Dec 位置进行beamform

    process_vcs.py -m beamform  -b [begin] -e [end] -o [obs ID] -p ["RA string" "DEC string"] --DI_dir [/path/to/DI_JonesMatrices*.dat/] --flagged_tiles [/path/to/flagged_tiles.txt] [--incoh]


# 后台数据传输
#### 用tmux窗口在后台下载galaxy数据
(https://gist.github.com/henrik/1967800)  
开启tmux `tmux`  

显示已有tmux列表 `tmux ls`  

临时退出session `Ctrl+b d`  

在session中翻页
>Ctrl+b \[  

退出翻页模式: `q`

横切split pane horizontal
>Ctrl+b ” (问号的上面，shift+’)

按顺序在pane之间移动
>Ctrl+b o

创建并指定session名字
>tmux new -s $session_name

进入已存在的session
>tmux a -t $session_name

删除指定session
>tmux kill-session -t $session_name  

删除当前session: `Ctrl+d`

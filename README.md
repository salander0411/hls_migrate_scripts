# 迁移HLS文件上s3

**HTTP Live Streaming**（缩写是**HLS**）是由苹果公司提出基于HTTP的流媒体网络传输协议。它的工作原理是把整个流分成一个个小的基于HTTP的文件来下载，每次只下载一些。当媒体流正在播放时，客户端可以选择从许多不同的备用源中以不同的速率下载同样的资源，允许流媒体会话适应不同的数据速率。在开始一个流媒体会话时，客户端会下载一个包含元数据的extended M3U (m3u8) playlist文件作为索引文件，用于寻找可用的媒体流。作为一种常见的互联网流媒体点播解决方案，很多播放器和软件都支持M3U文件格式。   

本文提供针对 HLS 文件迁移到 [S3](https://aws.amazon.com/cn/s3/) 的解决方案，其所含脚本将遍历 m3u8 索引文件的 list 以及其下所含的所有ts文件的链接，最终将所有 m3u8 文件和 ts 文件上传到 S3。

## 适用场景 

可用于上传到 S3 的迁移工具很多，比如[Amazon S3 断点续传工具](https://github.com/aws-samples/amazon-s3-resumable-upload)（支持本地上传S3，国内和海外S3互传，从阿里OSS迁移，支持多线程断点续传）, [rclone](https://rclone.org/s3/)（支持s3, Dreamhost, IBM等主流云盘互传）等等，针对 TB 级别以上的数据，AWS 还提供 [Snowball](https://aws.amazon.com/cn/snowball/) 来快速实现迁移。 

与这些解决方案相比，本文的使用场景偏向于以下情况。

1. 需要迁移的是 HLS 类型的文件。
1. 有一些从小众云盘迁移文件到 s3 的需求，是其他脚本工具所不支持的。
1. HLS 文件遍布在多个源服务器上，如 IDC 以及不同云盘。

此 python 脚本使用多线程设计，可以充分利用带宽与内存。且包含日志记录，校验脚本，一键重传等工具，避免出现个别 TS 分片因为网络原因出现上传失败导致文件无法播放的情况。

## Prerequisite 前置要求 | 准备工作

### 第一步. 整理m3u8链接列表
1. **推荐用txt文本保存m3u8链接链表**，video.txt 文件中每一行为一个完整的 m3u8 链接，如 http://abc.com/index.m3u8

### 第二步. 启动服务器。 
1. 启动 Amazon linux2 服务器，型号选择 ``t3.medium``, 打开 ``T2/T3 unlimited`` 的模式以充分利用资源。
1. 给 EC2 挂载具有完整 S3 权限的 IAM Role。或者在 CLI 上[配置AKSK密钥](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-configure.html) 

### 第三步.配置 python 环境
Linux 或 Windows上的 Python 3 环境 , 安装 requests 和 boto3 库。
#### Linux
   ```
   yum install python3  
   pip3 install requests 
   pip3 install boto3
   ```
#### Windows
   ```
   # download python3 from [python.org](https://www.python.org/downloads/windows/) 
   pip3 install requests
   pip3 install boto3
   ``` 

### 第四步. 准备脚本
1. 在迁移hls文件上云的过程中，我们主要需要4个脚本， _1_upload.py,  _2_check_error.py, 以及 _3_compare_ts.py, _4_re_upload_ts.py。 分别用于上传list，重传失败的文件，以及最后验证文件完整性, 以及再次重传没有的ts文件。先把这几个代码通过``git clone``拷贝到服务器上。

### 第五步. 确认S3路径
提前确认好需要上传至s3桶的 ``bucket`` 以及``prefix``。

## 使用步骤

## 针对每个txt列表 | 上传过程
1. 将 video_partN.txt 上传到服务器，并``cd``到对应目录
1. 打开 _1_upload.py。在“自修改信息区域”， 修改 ``file_path``，``bucket``，以及``prefix`` 这三个参数。这三个参数分别代表，本地 txt list 路径 ，目标 S3 桶，以及目标 S3 桶的prefix。 
**maxThreads经过测试 800 最好** 。
1. **请二次检查保证配置的信息是正确的**。
1. 在当前路径下执行 ``mkdir log``
1. 执行命令： ```nohup python3 _1_upload.py > log/video_partN.log 2>&1 &```
1. 去配置其他server，修改对应参数，可以同时并行多个server，比如10或者20个。因资源抢占问题，不建议在一个server上运行多个程序。
1. 过几个小时，登录到server上。查看进程有没有执行完毕。 ``ps aux | grep ‘python’``，如无，继续等待。不要删除或者移动任何文件。
1. 如执行完毕，检查实际处理的条数是否和video_partN.txt一致。执行 ``wc -l video_partN.txt``   ， 再执行  ``grep 'deal' log/video_partN.log | wc -l ``。 
   1. 如果两者数目一致，说明程序执行没有问题。ls可以发现文件夹多了一个日期的log，如2020-03-20-07-39-45.log，这个日志是记录 import.py 里由于超时等原因没有上传成功的url的。
    执行  ``python3 _2_check_error.py``,  该脚本会自动找到最新的日期log，将日期log 里面记录下来的没有上传成功的list重新上传。并且又会生成新的日期log。
     如果执行完 _2_check_error.py，  wc -l  最新的时间戳日志.log，看一下是不是0行，如果多于0行，继续执行 ``python3 _2_check_error.py`` （如果希望在后台运行不中断，使用nohup），直到没有新的错误link生成
   2. 如果不一致，说明实际程序执行到某一行时，可能因为特殊字符（中文或者是"）的原因自动退出了。这时候需要先执行 ``python3 _2_check_error.py`` ，再去对应的行数检查（vim进去之后，: set nu，然后到对应行查看）。
   3. 查看完毕解决字符问题后，重新建一个剩余url的list，修改 ``_1_upload.py`` 的参数，让 ``_1_upload.py`` 继续执行剩下的没有执行的行数。


## 等文件上传完毕 | 验证过程
1. 循环上述上传过程，直到某个二级域名下的所有数据都上传成功（即最新日期的日志没有错误链接了）。接下来，需要验证某个站是否都上传成功了，有没有漏掉的文件，
1. 打开 ``_3_compare_ts.py`` 这个文件，找到 自修改信息区域，修改bucket名字，prefix (如video-xxx-com/) 记得带 "/", txt_file为某个二级域名的全部txt list。（注意，一个站下有会有多个二级域名，要分开执行）
1. ``python3 _3_compare_ts.py`` ，该脚本会验证每个文件夹下的ts分片数是不是和index.m3u8当中的是一致的，并且记录那些不一致的URL到日志当中，日志为tsfile文件夹中的最新日期日志，路径：tsfile/2020-03-...log
1. 运行 ``python3 _4_re_upload_ts.py``,  修改bucket和prefix参数，把这些文件上传有漏的情况，重新上传一遍。
1. 二次运行 ``python3 _3_compare_ts.py``  ，检查日志是否为空，如果不为空，循环这个_4_re_upload_ts.py,直到最后日志为空。
   

##  脚本功能说明
### _1_upload.py
功能：<br>
1.给定txt的链接文件，扫描链接并完成上传到s3的操作<br>
2.404或502的链接将不会上传文件<br>
3.所有未成功的链接将记录到最新日期的log文件中（和_1_upload.py脚本同一路径）<br>
<br>
自定义脚本参数（可以自行修改变量）：<br>
file_path = ''  - txt文件链接，字符串类型，例如 'btt1.txt'<br>
maxThreads = 300  - 线程数量，int类型，例如300<br>
bucket = 'test--20200310'  - s3桶名称，字符串类型<br>
prefix = 'video1-hsanhl-com/'   - s3桶前缀路径，字符串类型  最前面不要/,最后要/，比如 'abc/123/'<br>

如果需要修改根据一级m3u8索引列出二级m3u8索引与其他ts文件链接的逻辑，请 在 **multi_thread()** 函数中修改。

### _2_check_error.py
功能：<br>
1.检索·upload.py·生成的最新日期的日志，并自动上传所有被记录的失败链接<br>
2.404的链接将不会上传文件<br>
3.所有未成功的链接将记录到最新日期的log文件中（和upload.py脚本同一路径）<br>
<br>
直接运行脚本即可<br>
可以通过修改 `upload.check_error(max_thread = 300)` 中的最大线程数来加快进度（受带宽和机器性能限制）<br>

### _3_compare_ts.py
功能：<br>
1.给定txt文件，分析所有s3中ts文件数量，和链接中应有文件数量不相等的url，记录到同路径tsfile文件夹下的最新日期日志文件中<br>
自定义脚本参数（可以自行修改变量）：<br>
my_bucket = s3.Bucket('test--20200310')  - 修改s3桶名称<br>
txt_file = 'btt1.txt'   - txt文件路径<br>
prefix = ''   - 前缀名称，不修改则会根据链接自动检测 - 字符串类型  最前面不要/,最后要/，比如 'abc/123/'<br>
<br>
<br>

### _4_re_upload_ts.py
功能：<br>
1.检索_3_compare_ts.py生成的最新日期日志，并上传所有ts文件<br>
2.失败的链接会在同路径下的最新日期日志文件中<br>
<br>

### upload_index.py
功能：<br>
1.根据给定的txt文件，上传所有index.m3u8文件（同ts文件同一目录）<br>
2.bucket和prefix请在upload.py文件中指定
3.上传失败的链接会记录到同文件夹下的最新日期日志中
<br>

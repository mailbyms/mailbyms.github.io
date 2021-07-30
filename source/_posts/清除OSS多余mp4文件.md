---
title: 清除OSS多余mp4文件
date: 2021-04-08 16:51:07
categories:
  - 运维
  - 脚本
tags:
  - shell
---

1. 以下场景用到视频文件，上传到 OSS 

   | 模块       | 数据库表          | 字段        | 说明                                                         |
   | ---------- | ----------------- | ----------- | ------------------------------------------------------------ |
   | 视频录制   | tiac_prod.live    | replay_url  | 只用来回调。腾讯回调，给出一个地址，会过期。<br>所以要上传到 OSS |
   | 云课堂作业 | homework_question | content_url | content_type为2时，表示视频                                  |
   |            | question          | video_url   |                                                              |
   |            | question_reply    | video_url   |                                                              |

   

2. 从数据库中导出视频文件列表（去掉域名），导出到 `video_files.txt`

   ```sql
SELECT REPLACE(r1, "https://oss-tiac.lconrise.cn", "") as r2
from
(SELECT REPLACE(replay_url, "http://oss-tiac.lconrise.cn", "") as r1
FROM `live`
WHERE replay_url is not NULL and is_delete=0) b
   ```

3. SHELL脚本遍历目录，找出所以mp4文件，如果不在 `video_files.txt` 里，则删掉

   ```shell
#!/bin/bash
#当前目录名
   top_dir=$(pwd)
   #把数据库查出来的文件名，加载到数组里
   array=($(awk '{print $1}' video_files.txt))

   #待删文件备份目录
   del_backup_dir="/data/nhykt_oss_backup_before_delete"
   mkdir -p "$del_backup_dir"

   #递归查找所有 mp4 文件，放到数组里
   mp4array=($(find . -name "*.mp4" -type f))
   for mp4 in ${mp4array[@]};
   do
   #因为find找出来的文件名，前面带"."，去掉第1个字符
     m=(${mp4:1})
     if [[ "${array[@]}" =~ "$m" ]]; then
       echo "$m exist: yes"
     else
       echo "$m exist: no"
       #用${m%/*}提取文件名的路径信息
       mkdir -p "$del_backup_dir/${m%/*}"
       #移动文件到待删备份目录
       mv ".$m" "$del_backup_dir/$m"
     fi
   done;
   ```



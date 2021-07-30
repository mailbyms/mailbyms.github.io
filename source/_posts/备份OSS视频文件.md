---
title: 备份OSS视频文件
date: 2021-04-28 16:51:07
categories:
  - 运维
  - 脚本
tags:
  - shell
---

1. 把 OSS 挂载到目录`/home/mike/nhykt-os`下，通过内网挂载

2. cron 脚本，每天凌晨2点0分执行

   ```
   0 2 * * * /data/oss_backup.sh >> /var/log/oss_backup.log
   ```

3. 备份脚本`oss_backup.sh`

   ```shell
   #!/bin/bash
   # Description: 备份 OSS 内容
   # Notes:      将脚本加入crontab中，每天定时执行
   MAX_LOOP_DAYS=1                      # 往前回顾检查的天数
   
   OSS_MOUNT_DIR=/home/mike/nhykt-oss           # OSS 挂载的根目录
   OSS_BACKUP_DIR=/data/nhykt_oss_backup/                  # 备份的目的地
   
   for ((i=$MAX_LOOP_DAYS; i>0; i--))
   do
     src_date=`date -d "$i days ago" +'%Y%m%d' `
     echo "`date` backup $src_date"
     cp -au $OSS_MOUNT_DIR/$src_date $OSS_BACKUP_DIR
     echo "`date` done"
   
   done
   ```

4. 通过 wget 直接下载备份

   从数据库查出完整URL列表，保存为`video_files.txt`

   ```sql
   SELECT replay_url
   FROM `live`
   WHERE replay_url is not NULL and is_delete=0
   ORDER BY replay_url
   ```

   或使用 cron 脚本，每天凌晨1点30分执行
   
   ```shell
   30 1 * * * /data/get_rds_video_file_list.sh >> /var/log/get_rds_video_file.log
   ```
获取视频文件列表脚本 `get_rds_video_file_list.sh`   
   
   ```shell
   #!/bin/bash
   
   OSS_BACKUP_DIR=/data/nhykt_oss_backup                  # 备份的目的地
   FILES_LIST="$OSS_BACKUP_DIR/video_files.txt"          # 视频文件列表，注意要使用 dos2unix 转换
   
   echo "$FILES_LIST"
   lastIdFile=/tmp/last_id.txt
   if [ ! -f "$lastIdFile" ]; then
     echo "lastIdFile 文件不存在，初始写入"
     echo "-1" > "$lastIdFile"
   fi
   
   # 用来记录上次获取视频文件URL（非下载）的最大ID
   lastId=$(cat $lastIdFile)
   echo "lastId: $lastId"
   
   FILES_LIST="video_files.txt"          # 视频文件列表，注意要使用 dos2unix 转换
   rds_host="rm-wz9m151k36pb70j9eo.mysql.rds.aliyuncs.com"
   user="tiac_prod"
   password="1Z%H7jx8PhG&#BQltOHyD4io"
   database="tiac_prod"
   
   echo "获取视频文件URL，写入文件${FILES_LIST}"
   # 查询 replay_url
   select_sql="SELECT replay_url FROM $database.live WHERE replay_url > '$lastId' and is_delete=0 ORDER BY replay_url"
   # 结果写入文件
   mysql -h"$rds_host" -u"$user" -p"$password" -e "${select_sql}" |awk 'NR>1' >> "$FILES_LIST" 
   
   echo "更新lastId"
   # 更新 last_id
   select_id_sql="SELECT max(replay_url) FROM $database.live WHERE replay_url is not NULL and is_delete=0 "
   mysql -h"$rds_host" -u"$user" -p"$password" -e "${select_id_sql}" |awk 'NR>1' > "$lastIdFile"
   
   ```
   
   
   
   备份脚本`oss_backup_wget.sh`
   
   ```shell
#!/bin/bash
   
   # Description: 备份 OSS 内容
   # Notes:      将脚本加入crontab中，每天定时执行
   
   MAX_LOOP=100                      # 每天备份的文件数
   
   OSS_BACKUP_DIR=/data/nhykt_oss_backup                  # 备份的目的地
   FILES_LIST="$OSS_BACKUP_DIR/video_files.txt"	      # 视频文件列表，注意要使用 dos2unix 转换
   
   for ((i=$MAX_LOOP; i>0; i--))
   do
   #读取文件第一行
     read line < $FILES_LIST  
     echo "`date -R` backup $line"
   
     file_name_full="$OSS_BACKUP_DIR/$(echo $line | cut -d/ -f4-)"
     file_name_subdir=${file_name_full%/*}
   
     echo "$file_name_full"
     echo "$file_name_subdir"
   
     #检查文件是否已存在，如果存在，与url大小对比
     if [ -f $file_name_full ]
     then
       localsize=$(stat --format=%s $file_name_full)
       remotesize=$(curl -s -I $line|grep Content-Length|awk '{print $2}'|tr -d "\r\n")
       # 检查文件大小，如果不相等，续传
       if [ "$localsize" = "$remotesize" ]
       then
         echo "file exists & good, contiune"
         sed -i '1d' $FILES_LIST
         continue
       else
         echo "file fragment found, continue download"
         cd $file_name_subdir
         wget -c $line
       fi
   # 文件不存在，
     else
       mkdir -p $file_name_subdir
       cd $file_name_subdir
       wget $line
     fi
   
     #删除文件第一行
     sed -i '1d' $FILES_LIST
   done
   
   
   ```

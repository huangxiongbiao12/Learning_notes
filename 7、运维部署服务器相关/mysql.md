## mysql安装



#### windows server

一般安装5.7版本

下载地址







Mysql  windows 备份

Sq_lbackup.bat

```
set "Ymd=%date:~,4%%date:~5,2%%date:~8,2%"
mysqldump brain_js -P7000 -h127.0.0.1 -uroot -pSskj@Rm#20220801 > E:\application\sql\data\brain_js_%Ymd%.sql
```


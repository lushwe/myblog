# Linux常用命令

## Linux系统相关
- 查看系统内核版本 -- `uname -r`
- 查看内存、CPU、磁盘使用情况 -- `top`
- 查看内存使用情况 -- `free` , `free -m` , `free -g`
- 查看磁盘使用情况 -- `du` , `df`
- 查看进程、端口情况 -- `netstat -tunlp | grep pid/port`
- 查看Java进程 -- `ps -ef|grep java`
- 强制杀进程 -- `kill -9 pid`

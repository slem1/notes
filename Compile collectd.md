
# Compile collecd


```
yum install python2-devel autoconf automake flex bison libtool pkg-config
./build.sh
./configure --enable-debug --enable-python --enable-ping $OTHER_FLAGS CFLAGS="-g -O0" | tee configure.log
make install 
```


`/opt/collectd/sbin/collectd -C /opt/collectd/etc/collectd.conf`



# logfile plugin with debug

```
    LoadPlugin "logfile"
    <Plugin "logfile">
    LogLevel "DEBUG"
    File "/var/log/collectd/collectd.log"
    Timestamp true
    PrintSeverity "true"
    </Plugin>
```


# gdb 

`gdb /opt/sbin/collectd <PID> | <core_dump_file>`


debuginfo-install libgcc-4.8.5-44.el7.x86_64 libstdc++-4.8.5-36.el7_6.2.x86_64 python-libs-2.7.5-80.el7_6.x86_64 python2-snappy-0.5-8.el7.x86_64


- check missing debugsymnbols
```
set verbose on
run
```


info frame
frame <#number>
bt

# Force a core dump
as root

`ulimit -c unlimited && kill -6 <PID>` 


# Force log rotate

`logrotate /etc/logrotate.d/collectd`

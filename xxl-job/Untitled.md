-Xmx4g -Xms4g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -Xss256k -XX:+UseG1GC -XX:SurvivorRatio=5 -XX:+AlwaysPreTouch -XX:MaxGCPauseMillis=100 -XX:G1ReservePercent=25 -XX:-U
seLargePages -XX:+ParallelRefProcEnabled -XX:InitiatingHeapOccupancyPercent=40 -XX:+ExplicitGCInvokesConcurrent -XX:ParallelGCThreads=8 -XX:ConcGCThreads=4 -XX:PretenureSizeThreshold=1m -XX:-Use
BiasedLocking -XX:AutoBoxCacheMax=20000 -XX:MaxTenuringThreshold=6 -Xloggc:/dev/shm/gc.log -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintCodeCache -XX:+
UseGCLogFileRotation -XX:NumberOfGCLogFiles=2 -XX:GCLogFileSize=10m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/root/logs/ -XX:MaxDirectMemorySize=1G -XX:-OmitStackTraceInFastThrow

317 Jstat -Xmx4g -Xms4g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -Xss256k -XX:+UseG1GC -XX:SurvivorRatio=5 -XX:+AlwaysPreTouch -XX:MaxGCPauseMillis=100 -XX:G1ReservePercent=25 -XX:-UseLa
rgePages -XX:+ParallelRefProcEnabled -XX:InitiatingHeapOccupancyPercent=40 -XX:+ExplicitGCInvokesConcurrent -XX:ParallelGCThreads=8 -XX:ConcGCThreads=4 -XX:PretenureSizeThreshold=1m -XX:-UseBias
edLocking -XX:AutoBoxCacheMax=20000 -XX:MaxTenuringThreshold=6 -Xloggc:/dev/shm/gc.log -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintCodeCache -XX:+UseG
CLogFileRotation -XX:NumberOfGCLogFiles=2 -XX:GCLogFileSize=10m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/root/logs/ -XX:MaxDirectMemorySize=1G -XX:-OmitStackTraceInFastThrow -Dapplicati
on.home=/usr/java/jdk1.8.0_161 -Xms8m





MaxGCPauseMillis

ParallelGCThreads



ConcGCThreads



G1ReservePercent



ParallelRefProcEnabled
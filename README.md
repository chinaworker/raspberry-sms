# 树莓派+EC20短信转发到企业微信

一、材料准备：
1.树莓派
2.4G模块 （比如ec20，ec20兼容性强，推荐）

#注册企业微信
1.首先去下面链接注册企业微信
https://work.weixin.qq.com/wework_admin/register_wx

2.认证需要花钱，无需认证无需认证无需认证

3.注册完成后，新建应用。步骤：“应用管理”->“自建”->“创建应用”

4.完成以上步骤后，拿到以下三个关键要素
  企业ID：“我的企业”->“企业信息”->页面下方有个“企业ID”

   AgentId和Secret：“应用管理”->点击对应应用程序进入
  
二、部署过程

1.把“pushToWecha.py”和“smsForward.py”放入“/home/pi/tools”下（注意用户、组和权限）
2.修改“pushToWechat.py” 中的
   #--------------------获取企业微信配置信息--------------------
    #企业ID
    corpID  = 'replace' #替换自己的企业ID
    #应用密钥
    secret  = 'replace' #替换自己的密钥
    #应用ID
    agentID = 'replace'#替换自己的应用ID
    
 3.把sim卡插入EC20模块，把模块插入树莓派的USB。使用以下命令可以看到有 ttyUSB0~3四个设备，应该对应EC20不同的口，比如短信口或者数据口：
  pi@raspberrypi:~ $ lsusb
  Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
  Bus 001 Device 003: ID 2c7c:0125 Quectel Wireless Solutions Co., Ltd. EC25 LTE modem
  Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
  Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
  
  pi@raspberrypi:~ $ ls /dev/ttyUSB*
  /dev/ttyUSB0  /dev/ttyUSB1  /dev/ttyUSB2  /dev/ttyUSB3
  
4.安装和配置gammu
  4.1安装gammu
  sudo apt-get install gammu
  
  4.2配置gammu
  sudo gammu-config
                                                                      │
     │ Configuration file "/root/.gammurc" exists.                        │
     │ Do you still wish to configure Gammu?                              │
     │                                                                    │
     │                  <是>                      <否>                    
     
     
     ┌────────────────────────────────────────────────────────────────────┐
     │                                                                    │
     │ Configuration file "/root/.gammurc" exists.                        │
     │ Do you wish to read defaults from it?                              │
     │                                                                    │
     │                  <是>                      <否>                    │
     │                                                                    │
     └────────────────────────────────────────────────────────────────────┘

     
  直接两个<是>跳过
  
  按照下面配置，保存后退出：
  
  
                  │ Current Gammu configuration             │
                  │                                         │
                  │  P Port                 (/dev/ttyUSB2)  │
                  │  C Connection           (at19200)       │
                  │  M Model                ()              │
                  │  D Synchronize time     (yes)           │
                  │  F Log file             ()              │
                  │  O Log format           (nothing)       │
                  │  L Use locking          ()              │
                  │  G Gammu localisation   ()              │
                  │  H Help                                 │
                  │  S Save                                 │
                  │                                         │
                  │                                         │
                  │        <确定>          <取消>           │
                  │                                         │
                  └─────────────────────────────────────────┘
    
   4.3测试是否配置成功
      查看设备信息
      
       sudo gammu --identify
       
       pi@raspberrypi:~ $ sudo gammu --identify
       设备               : /dev/ttyUSB2
       制造商            : Quectel
       型号               : unknown (EC20F)
       固件               : EC20CEFAGR06A15M4G
       IMEI                 : 888888888
       SIM IMSI             : 888888
       
   尝试发送短信
   
   使用以下命令尝试发送短信（后面的186XXXXXXXX自行替换成接收短信的手机号），如果能收到短信，则说明配置成功，如果报错350，买天线吧
   
   echo "a test sms from ec20" | sudo gammu sendsms TEXT 186XXXXXXXX
   
  5.安装和配置gammu-smsd，用于接收短信
  
    5.1 安装gammu-smsd
    
    sudo apt-get install gammu-smsd
    
    5.2 配置 gammu-smsd
    sudo nano /etc/gammu-smsdrc
     把gammu-smsdrc这个文件修改下
     
      # Configuration file for Gammu SMS Daemon

      # Gammu library configuration, see gammurc(5)

       [gammu]

     # Please configure this!

     port = /dev/ttyUSB3

     #port = /dev/ec20

     connection = at19200

    # Debugging

    #logformat = textall

    # SMSD configuration, see gammu-smsdrc(5)

    [smsd]

    service = files

    RunOnReceive = /usr/bin/python3 /home/pi/tools/smsForward.py

    #logfile = syslog

    logfile = /home/pi/gammu/log/log_smsd.log

    # Increase for debugging information

    debuglevel = 0

    # Paths where messages are stored

    inboxpath = /home/pi/gammu/inbox/

    outboxpath = /home/pi/gammu/outbox/

    sentsmspath = /home/pi/gammu/sent/

    errorsmspath = /home/pi/gammu/error/
    
    
 关键配置项简单说明：

port：对应上面配置的设备口

service：接收后短信在本地的保存方式，这里配置为files，表示收到的短信以txt文本方式存放，当然也支持将短信存入数据库

RunOnReceive：短信收到后调用什么脚本或程序，这里是短信内容拼接后转发微信的程序，也就是“smsForward.py”

inboxpath/outboxpath/sentsmspath/errorsmspath：短信收件箱发件箱的路径，这里只关心inboxpath，收到的短信放在该路径下，文件名”IN<date>_<time>_<serial>_<sender>_<sequence>.txt“
    
   
   5.3 手工启动gammu-smsd，并尝试接收短信 
   
   sudo gammu-smsd --config /etc/gammu-smsdrc --pid /var/run/gammu-smsd.pid --daemon --user pi --group pi
   
   
   5.4 配置 gammu-smsd开机自启动
   sudo nano /lib/systemd/system/gammu-smsd.service
   
   贴入以下内容：
   
    [Unit]

    Description=SMS daemon for Gammu

    Documentation=man:gammu-smsd(1)

    After=mysql.service postgresql.service

    [Service]

    EnvironmentFile=-/etc/sysconfig/gammu-smsd

    # Run daemon as root user

    ExecReload=/bin/kill -HUP $MAINPID

    ExecStopPost=/bin/rm -f /var/run/gammu-smsd.pid

    Type=forking

    PIDFile=/var/run/gammu-smsd.pid

    ExecStartPre=/bin/sleep 30

    ExecStart=/usr/bin/gammu-smsd --config /etc/gammu-smsdrc --pid /var/run/gammu-smsd.pid --daemon --user pi --group pi

    [Install]

    WantedBy=multi-user.target
  
关键配置项简单说明：

ExecStartPre：树莓派启动后，EC20设备会初始化，为了避免gammu-smsd启动先于设备初始化，这里做一个30秒的等待

通过以下命令开启开机自启动：

     sudo systemctl enable gammu-smsd
 gammu-smsd的启停命令：
 
     sudo systemctl start gammu-smsd

     sudo systemctl stop gammu-smsd

     sudo systemctl restart gammu-smsd
     
 5.5 针对gammu-smsd的关机优化脚本
 
   实践发现EC20这块卡有个问题，树莓派系统关机前如果gammu-smsd进程还在的话，下次启动时EC20短信口因为被占用，ttyUSB编号会变，比如从ttyUSB3变成ttyUSB2，使得配置失效，所以关机前需要把gammu-smsd进程kill掉
     
      sudo nano /lib/systemd/system-shutdown/kill-gammu-smsd.sh
      
     写入以下内容:
     
      #!/bin/bash

      ps aux | grep gammu-smsd | grep -v 'grep' | awk '{print $2}' | xargs kill -9
  

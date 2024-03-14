---
layout: post
title: Linux 自动测试
date: 2023-12-18 10:16 -0500
image:
  path: /assets/img/blog/linux-auto-test/architecture.png
description: >
  如何搭建一个嵌入式Linux的自动测试平台？
---
# Linux 自动测试

## 目录

-   [目标结构](#目标结构)
-   [安装](#安装)
-   [创建工程](#创建工程)
-   [配置目标板](#配置目标板)
-   [编写测试用例](#编写测试用例)
-   [gitlab ci 集成](#gitlab-ci-集成)
-   [运行效果](#运行效果)

最近需要在嵌入式Linux板上搭建一套测试平台，需要满足如下需求：

-   需要运行一些如i2c\_tools等命令，测试硬件的工作状态。
-   可以控制外部电源做冷启动。
-   可以将运行结果返回给服务器。
-   方便集成到gitlab中，利用ci指定运行时间或次数。
-   收集测试日志，测试结果，生成report。

然后发现了这样一个Python开发的，基本可以满足上述需求的，针对嵌入式Linux的自动测试工具：[tbot](https://tbot.tools/ "tbot")。本文将简要介绍如何利用tbot搭建一套测试平台以满足上述需求。

## 目标结构

![](/assets/img/blog/docker/architecture.png)

官方这里给出了一个预设的结构，基本可以满足目前的需求，其中：

-   tbot host：用于执行tbot的命令，同时gitlab也搭建在这个服务器上，最后会将tbot的命令通过gitlab的ci运行。
-   Lab Host：由于服务器在机房，不太方便插板子，因此搞了一个ubuntu台式机连到内网，板子插到了这台台式机上。
-   Board：是这次的DUT（device under test）
-   Buildhost：目前没有用到，由于一般嵌入式Linux可以在uboot阶段通过tftp加载kernel镜像并挂载文件系统，因此后续可以设置一个编译服务器，将镜像编译出来加载到板子上，并将这部分功能加入到gitlab中。

## 安装

安装步骤省略介绍，详情看官方文档。

## 创建工程

按照如下目录结构新建文件

```bash
./
├──config/
│  └──board_config.py # 板子相关的配置
└──tc/
│  ├──test_linux.py # linux测试用例
│  ├──test_uboot.py # uboot测试用例
│  └──interactive.py # 用来访问板子上的shell
└──conftest.py # pytest的配置文件


```

## 配置目标板

```python
# board_config.py
import tbot
import time
from tbot.machine import board, connector, linux


class Board_EVB(connector.ConsoleConnector, board.PowerControl, board.Board):
    ''' board 的基本配置 '''
    baudrate = 115200
    serial_port = "/dev/ttyUSB0"

    def connect(self, mach: linux.LinuxShell):

        return mach.open_channel('picocom', '-b', str(self.baudrate), self.serial_port)

    def poweron(self) -> None:

        tbot.log.message('Power on')

        # 1. 如果有程控电源，可以在labhost上执行对应的命令给目标板上电。
        # with tbot.ctx.request(tbot.role.LocalHost) as lo:
        #     lo.exec0('sispmctl')

        # 2. 如果没有程控电源，目标板是Linux，通过reboot命令重启。
        time.sleep(0.1)
        # 发送ctrl-c，结束当运行命令。
        self.ch.sendcontrol('C')
        time.sleep(0.1)
        self.ch.sendline('reboot')

        # 3. 也使用U-Boot，也可以从uboot重启。“cat”用于确保我们不会将“reset”发送到Linux，因为这会干扰终端。。。
        self.ch.sendline("cat")
        self.ch.sendline("reset")
    
    def poweroff(self) -> None:
        with tbot.ctx.request(tbot.role.LocalHost) as lo:
            tbot.log.message('Power off')

class Board_EVB_Linux(board.Connector, board.LinuxBootLogin, linux.Ash):
    username = 'root'  # linux bord 的用户名
    password = '123456' # linux board 的密码

    def init(self):
        ''' 可以做一些初始化，为后面测试做准备 '''

        pass

class Board_EVB_Uboot(board.Connector, board.UBootAutobootIntercept, board.UBootShell):
    prompt = '=> ' # u-boot 终端提示符

class Server(connector.SSHConnector, linux.Bash):
    hostname = '10.8.32.196'
    username = 'admin'
    port = 22
    ignore_hostkey = True
    authenticator = linux.auth.PasswordAuthenticator("123456")




def register_machines(ctx):
    ''' 将上述 class 注册为各种role，后续编写测试用例会用到 '''
    ctx.register(Board_EVB, tbot.role.Board)
    ctx.register(Board_EVB_Linux, tbot.role.BoardLinux)
    ctx.register(Board_EVB_Uboot, tbot.role.BoardUBoot)

    ctx.register(Server, tbot.role.LabHost)


```

## 编写测试用例

```python
# interactive.py
import tbot

@tbot.testcase
def board() -> None:
    """Open an interactive session on the board's serial console."""
    with tbot.ctx.request(tbot.role.Board) as b:
        b.interactive()

@tbot.testcase
def linux() -> None:
    """Open an interactive session on the board's Linux shell."""
    with tbot.ctx.request(tbot.role.BoardLinux) as lnx:
        lnx.interactive()

@tbot.testcase
def uboot() -> None:
    """Open an interactive session on the board's U-Boot shell."""
    tbot.ctx.teardown_if_alive(tbot.role.BoardLinux)
    with tbot.ctx.request(tbot.role.BoardUBoot) as ub:
        ub.interactive()


```

```python
# test_linux.py
from time import sleep, time
import tbot

DATA_DIR='/data'

# 由于后续会与pytest结合，生成xml和html格式测试报告，
# 因此测试用例的函数名需要以test开头，以便pytest可以识别到

@tbot.testcase
def test_setup():
    with tbot.ctx.request(tbot.role.BoardLinux) as lnx:
        tbot.log.message('Set up the testing environment')
        lnx.exec0('mkdir', '-p', f'{DATA_DIR}')
        lnx.exec0('mount', '/dev/mmcblk0p1', '/data')
        lnx.ch.sendcontrol('M')
        
@tbot.testcase
def test_norflash_read():
    with tbot.ctx.request(tbot.role.BoardLinux) as lnx:
        tbot.log.message('Start testing norflash')
        lnx.exec0('fio', '-filename=/dev/mtdblock1', '-direct=1',
                  '-iodepth=1', '-thread', '-rw=read', '-ioengine=psync', 
                  '-bs=1M', '-size=10M', '-numjobs=1', '-runtime=1', '-group_reporting',
                  '-name=file')
        sleep(5)
        
@tbot.testcase
def test_norflash_write():
    with tbot.ctx.request(tbot.role.BoardLinux) as lnx:
        tbot.log.message('Start testing norflash')
        lnx.exec0('fio', '-filename=/dev/mtdblock1', '-direct=1',
                  '-iodepth=1', '-thread', '-rw=write', '-ioengine=psync', 
                  '-bs=1M', '-size=10M', '-numjobs=1', '-runtime=1', '-group_reporting',
                  '-name=file')
        sleep(5)



# 省略后续测试用例
```

```python
# conftest.py
import pytest
import tbot
from tbot import newbot

def pytest_addoption(parser, pluginmanager):
    parser.addoption("--tbot-config", action="append", default=[], dest="tbot_configs")
    parser.addoption("--tbot-flag", action="append", default=[], dest="tbot_flags")

def pytest_configure(config):
    # Only register configuration when nobody else did so already.
    if not tbot.ctx.is_active():
        # Register flags
        for tbot_flag in config.option.tbot_flags:
            tbot.flags.add(tbot_flag)
        # Register configurations
        for tbot_config in config.option.tbot_configs:
            newbot.load_config(tbot_config, tbot.ctx)

@pytest.fixture(scope="session", autouse=True)
def tbot_ctx(pytestconfig):
    with tbot.ctx:
        # Configure the context for keep_alive (so machines can be reused
        # between tests).  reset_on_error_by_default will make sure test
        # failures lead to a powercycle of the DUT anyway.
        with tbot.ctx.reconfigure(keep_alive=True, reset_on_error_by_default=True):
            # Tweak the standard output logging options
            with tbot.log.with_verbosity(tbot.log.Verbosity.STDOUT, nesting=1):
                yield tbot.ctx

```

## gitlab ci 集成

```yaml
stages:          # List of stages for jobs, and their order of execution
  - test

variables:
  REPEAT_COUNT:
    value: "5"
    description: "The number of repetitions of test cases"



tbot-test-job:   # This job runs in the test stage.
  stage: test    # It only starts when the job in the build stage completes successfully.
  tags:
    - test
  when: manual
  artifacts:
    paths:
      - report
    when: always
    reports: 
      junit: report/report.xml
  script:
    - source /home/chenyu.wang/.venv/bin/activate
    - mkdir report
    # - newbot -c config.board_dr1x90_ad101_v10 tc.test_linux.test_norflash_read
    - pytest --tbot-config config.board_dr1x90_ad101_v10 -vs --count=$REPEAT_COUNT --junitxml="./report/report.xml" ./tc/test_linux.py | tee report/test.log

```

## 运行效果

![](/assets/img/blog/docker/summary.png)

![](/assets/img/blog/docker/test_report.png)

```bash
============================= test session starts ==============================
platform linux -- Python 3.8.16, pytest-7.4.0, pluggy-1.2.0 -- /home/chenyu.wang/.venv/bin/python
cachedir: .pytest_cache
rootdir: /var/lib/gitlab-runner/builds/1y6pz3-N/0/embedded-sw/tbot
collecting ... collected 11 items

tc/test_linux.py::test_setup │   ├─Calling test_setup ...
│   │   ├─[local] sshpass -p 123456 ssh -o StrictHostKeyChecking=no -p 22 anlogic@10.8.32.196
│   │   ├─[local] sshpass -p 123456 ssh -o StrictHostKeyChecking=no -p 22 anlogic@10.8.32.196
│   │   ├─[server] picocom -b 115200 /dev/ttyUSB0
│   │   ├─POWERON (board_dr1x90_ad101_v10)
│   │   ├─Power on
│   │   ├─LINUX (board_dr1x90_ad101_v10_-linux)
│   │   │    <> picocom v3.1
│   │   │    <> 
│   │   │    <> port is        : /dev/ttyUSB0
│   │   │    <> flowcontrol    : none
│   │   │    <> baudrate is    : 115200
│   │   │    <> parity is      : none
│   │   │    <> databits are   : 8
│   │   │    <> stopbits are   : 1
│   │   │    <> escape is      : C-a
│   │   │    <> local echo is  : no
│   │   │    <> noinit is      : no
│   │   │    <> noreset is     : no
│   │   │    <> hangup is      : no
│   │   │    <> nolock is      : no
│   │   │    <> send_cmd is    : sz -vv
│   │   │    <> receive_cmd is : rz -vv -E
│   │   │    <> imap is        : 
│   │   │    <> omap is        : 
│   │   │    <> emap is        : crcrlf,delbs,
│   │   │    <> logfile is     : none
│   │   │    <> initstring     : none
│   │   │    <> exit_after is  : not set
│   │   │    <> exit is        : no
│   │   │    <> 
│   │   │    <> Type [C-a] [C-h] to see available commands
│   │   │    <> Terminal ready
│   │   │    <> ^C
│   │   │    <> [36mboard_dr1x90_ad101_v10_-linux: [32m~[0m> reboot
│   │   │    <> [36mboard_dr1x90_ad101_v10_-linux: [32m~[0m> Stopping linuxptp system clock synchronization: OK
│   │   │    <> Stopping linuxptp daemon: start-stop-daemon: warning: killing process 153: No such process
│   │   │    <> FAIL
│   │   │    <> Stopping sshd: OK
│   │   │    <> Stopping network: OK
│   │   │    <> Saving random seed: OK
│   │   │    <> Stopping klogd: OK
│   │   │    <> Stopping syslogd: OK
│   │   │    <> umount: devtmpfs busy - remounted read-only
│   │   │    <> umount: can't unmount /: Invalid argument
│   │   │    <> The system is going down NOW!
│   │   │    <> Sent SIGTERM to all processes
│   │   │    <> 
│   │   │    <> Sent SIGKILL to all processes
│   │   │    <> [59808.009015] reboot: Restarting system
│   │   │    <> DDR Init V3
│   │   │    <> [DDR GPLL] fck = 533.333 MHz, fbk_div = 64
│   │   │    <> [DDR PPC] Enable Lane Mask = 0xF
│   │   │    <> [DDR MDL CAL] DX0 mdl = 0x1270127
│   │   │    <> [DDR MDL CAL] DX1 mdl = 0x1270129
│   │   │    <> 
│   │   │    <> [DDR MDL CAL] DX2 mdl = 0x1290129
│   │   │    <> [DDR MDL CAL] DX3 mdl = 0x1290129
│   │   │    <> [DDR DCC] #0 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #1 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #2 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #3 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #4 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #5 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #6 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #7 initial dutyx = 0x1f
│   │   │    <> [DDR DCC] #0 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #1 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #2 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #3 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #4 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #5 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #6 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR DCC] #7 pol = 0x0, dutyx = 0x1f, overflow = 0x0, undeflow = 0
│   │   │    <> [DDR MRS] DDR3 Mode
│   │   │    <> [DDR MRS] MR0 = 0x950
│   │   │    <> [DDR MRS] MR1 = 0x44
│   │   │    <> 
│   │   │    <> [DDR MRS] MR2 = 0x210
│   │   │    <> [DDR MRS] MR3 = 0x0
│   │   │    <> [DDR WL] DX0GSR0.WLPRD = 0
│   │   │    <> [DDR WL] Done
│   │   │    <> [DDR GATE] DX0GSR0.GDQSPRD = 146
│   │   │    <> [DDR GATE] Done
│   │   │    <> [DDR WLADJ] Done
│   │   │    <> ========================================
│   │   │    <> DX DGSL WLSL WLD WDQD DQSGD RDQSD RDQSND
│   │   │    <> ----------------------------------------
│   │   │    <>  0    3    2  16   73    65    72     73
│   │   │    <>  1    3    2  21   73    58    72     73
│   │   │    <>  2    3    2  43   73    79    73     73
│   │   │    <>  3    3    2  40   74    83    72     73
│   │   │    <> ========================================
│   │   │    <> [DDR TRAIN #0] RDDSKE:O WRDSKW:X RDEYE:X WREYE:X
│   │   │    <> [DDR TRAIN #1] RDDSKE:O WRDSKW:O RDEYE:O WREYE:O
│   │   │    <> ========================================
│   │   │    <> DX DGSL WLSL WLD WDQD DQSGD RDQSD RDQSND
│   │   │    <> ----------------------------------------
│   │   │    <>  0    3    2  17   37    86   107     96
│   │   │    <>  1    3    2  22   40    82   108     99
│   │   │    <>  2    3    2  44   41   101   109     95
│   │   │    <>  3    3    2  41   37   104   108     97
│   │   │    <> ========================================
│   │   │    <> 
│   │   │    <> [DDR MTEST] #0x0, bistDone = 0x1, DxErr = 0x0, rcvCnt = 117
│   │   │    <> [DDR MTEST] #0x1, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x2, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x3, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x4, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x5, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x6, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x7, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x8, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x9, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0xa, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0xb, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0xc, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0xd, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0xe, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0xf, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x10, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x11, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x12, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x13, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x14, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x15, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x16, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x17, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x19, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x1a, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> [DDR MTEST] #0x1b, bistDone = 0x1, DxErr = 0x0, rcvCnt = 128
│   │   │    <> ==================== In Stage 1 ====================
│   │   │    <> System reset
│   │   │    <> PMU Error Config Init
│   │   │    <> PMU Error Config Init
│   │   │    <> history reset reason: 00000002
│   │   │    <> history pmu status 0: 00000000
│   │   │    <> history pmu status 1: 00000000
│   │   │    <> mark fsbl is running...
│   │   │    <> ==================== In Stage 2 ====================
│   │   │    <> Boot Mode: 0x00000005
│   │   │    <> EMMC or SD Boot Mode
│   │   │    <> multi boot offset is 0
│   │   │    <> 
│   │   │    <> Response: 0x900
│   │   │    <> CMD 1 send err, code: 0x18000
│   │   │    <> Bus width is 4-bit
│   │   │    <> ----------Card Info----------
│   │   │    <> |-Device: mmc@0xf8049000
│   │   │    <> |-Manufacturer ID: 0x3
│   │   │    <> |-OEM: 0x5344
│   │   │    <> |-Name: SL16G
│   │   │    <> |-Bus Speed: 50000 KHz
│   │   │    <> |-Mode: High Speed
│   │   │    <> |-Rd Block Len: 512
│   │   │    <> |-Version: 3
│   │   │    <> |-High Capacity: YES
│   │   │    <> |-Capacity: 14.8GB
│   │   │    <> |-Bus Width: 4-bit
│   │   │    <> -----------------------------
│   │   │    <> drv is sd
│   │   │    <> file name is BOOT.bin
│   │   │    <> FsblInstancePtr->ImageOffsetAddress: 0x0
│   │   │    <> boot header copy finished...
│   │   │    <> Calculated Checksum: 0x344c7492
│   │   │    <> checksum check pass...
│   │   │    <> partition headers copy finished...
│   │   │    <> image header authentication not enabled....
│   │   │    <> qspi width sel        : 0x00000000
│   │   │    <> image id              : 0x43474c41
│   │   │    <> enc status            : 0x00000000
│   │   │    <> bh attribute          : 0x00000000
│   │   │    <> bh ac offset          : 0x00000000
│   │   │    <> first parti hdr offset: 0x00000300
│   │   │    <> partition num         : 0x00000003
│   │   │    <> bh checksum           : 0x344c7492
│   │   │    <> ==================== In Stage 3 ====================
│   │   │    <> wdt restart
│   │   │    <> Partition Header Checksum: 0x0f9502a6
│   │   │    <> Calculated Checksum: 0x0f9502a6
│   │   │    <> checksum check pass...
│   │   │    <> Hash Type: 0x00000800
│   │   │    <> Hash SHA256.
│   │   │    <> Auth Type       : 0
│   │   │    <> Efuse Auth Type : 0
│   │   │    <> Authentication NOT enabled...
│   │   │    <> Enc Type       : 0
│   │   │    <> Efuse Enc Type : 0
│   │   │    <> Encrypt NOT enabled.
│   │   │    <> destination cpu: 0x00000020
│   │   │    <> destination dev: 0x00000080
│   │   │    <> check partition length: 
│   │   │    <> 
│   │   │    <> Partition Length            : 0x002bec40
│   │   │    <> Extracted Partition Length  : 0x002bec08
│   │   │    <> Total Partition Length      : 0x002bec80
│   │   │    <> Next Partition Header Offset: 0x00000380
│   │   │    <> Dest Execution Address      : 0xffffffff
│   │   │    <> Dest Load Address           : 0x200000
│   │   │    <> Partition Offset            : 0x00025200
│   │   │    <> Partition Attribute         : 0x000008a0
│   │   │    <> Hash Data offset            : 0x002e3e40
│   │   │    <> AC Offset                   : 0x00000000
│   │   │    <> Partition Header Checksum   : 0x0f9502a6
│   │   │    <> 
│   │   │    <> loading pl partition...
│   │   │    <> PCAP RESET
│   │   │    <> Re-enable PCAP
│   │   │    <> Boot device block size max: 4000
│   │   │    <> PL init done
│   │   │    <> cfg state: 00000001
│   │   │    <> Auth type :  00
│   │   │    <> Hash type :  02
│   │   │    <> Enc type  :  65
│   │   │    <> Enc mode  :  00
│   │   │    <> Key mode  :  6e
│   │   │    <> to ddr in a whole block, then to pcap
│   │   │    <> partition src address      : 0x00025200
│   │   │    <> partition load dest address: 0x00100000
│   │   │    <> partition length           : 0x002bec40
│   │   │    <> whole block
│   │   │    <> calculate hash
│   │   │    <> read hash
│   │   │    <> Hash check passed...
│   │   │    <> 
│   │   │    <> cfg state before progdone: 00000007
│   │   │    <> ==================== In Stage 3 ====================
│   │   │    <> wdt restart
│   │   │    <> Partition Header Checksum: 0xbcb16562
│   │   │    <> Calculated Checksum: 0xbcb16562
│   │   │    <> checksum check pass...
│   │   │    <> Hash Type: 0x00000800
│   │   │    <> Hash SHA256.
│   │   │    <> Auth Type       : 0
│   │   │    <> Efuse Auth Type : 0
│   │   │    <> Authentication NOT enabled...
│   │   │    <> Enc Type       : 0
│   │   │    <> Efuse Enc Type : 0
│   │   │    <> Encrypt NOT enabled.
│   │   │    <> destination cpu: 0x00000020
│   │   │    <> destination dev: 0x00000000
│   │   │    <> check partition length: 
│   │   │    <> 
│   │   │    <> Partition Length            : 0x000c6f40
│   │   │    <> Extracted Partition Length  : 0x000c6f32
│   │   │    <> Total Partition Length      : 0x000c6f80
│   │   │    <> Next Partition Header Offset: 0x00000000
│   │   │    <> Dest Execution Address      : 0x200000
│   │   │    <> Dest Load Address           : 0x200000
│   │   │    <> Partition Offset            : 0x002e3e80
│   │   │    <> Partition Attribute         : 0x00000820
│   │   │    <> Hash Data offset            : 0x003aadc0
│   │   │    <> AC Offset                   : 0x00000000
│   │   │    <> Partition Header Checksum   : 0xbcb16562
│   │   │    <> 
│   │   │    <> loading ps partition...
│   │   │    <> partition src address      : 0x2e3e80
│   │   │    <> partition load dest address: 0x200000
│   │   │    <> partition length           : 0x000c6f40
│   │   │    <> Auth type :  00
│   │   │    <> Hash type :  02
│   │   │    <> Enc type  :  65
│   │   │    <> Enc mode  :  00
│   │   │    <> Key mode  :  6e
│   │   │    <> calculate hash
│   │   │    <> Hash check passed...
│   │   │    <> ps partition copy finished
│   │   │    <> Update handoff values
│   │   │    <> ==================== In Stage 4 ====================
│   │   │    <> Mark FSBL is completed...
│   │   │    <> wdt disable
│   │   │    <> 
│   │   │    <> 
│   │   │    <> U-Boot 2021.01-g6520da2793 (Mar 04 2024 - 16:17:10 +0800)
│   │   │    <> 
│   │   │    <> CPU:   Anlogic dr1m90
│   │   │    <> Model: Anlogic dr1m90 demo board
│   │   │    <> DRAM:  1 GiB
│   │   │    <> SF: Detected n25q128a11 with page size 256 Bytes, erase size 4 KiB, total 16 MiB
│   │   │    <> MMC:   mmc@f8049000: 0
│   │   │    <> Loading Environment from FAT... *** Warning - bad CRC, using default environment
│   │   │    <> 
│   │   │    <> In:    serial@f8401000
│   │   │    <> Out:   serial@f8401000
│   │   │    <> Err:   serial@f8401000
│   │   │    <> Model: Anlogic dr1m90 demo board
│   │   │    <> Net:   eth0: ethernet@F8100000
│   │   │    <> Hit any key to stop autoboot:  3  2  1  0 
│   │   │    <> switch to partitions #0, OK
│   │   │    <> mmc0 is current device
│   │   │    <> ** Unrecognized filesystem type **
│   │   │    <> Scanning mmc 0:1...
│   │   │    <> Found U-Boot script /boot.scr
│   │   │    <> 472 bytes read in 16 ms (28.3 KiB/s)
│   │   │    <> ## Executing script at 13000000
│   │   │    <> 
│   │   │    <> Partition Map for MMC device 0  --   Partition Type: DOS
│   │   │    <> 
│   │   │    <> Part  Start Sector  Num Sectors  UUID    Type
│   │   │    <>   1  8192        31108096    00000000-01  0c
│   │   │    <> 7572921 bytes read in 336 ms (21.5 MiB/s)
│   │   │    <> 11969060 bytes read in 518 ms (22 MiB/s)
│   │   │    <> 16281 bytes read in 20 ms (794.9 KiB/s)
│   │   │    <> ## Booting kernel from Legacy Image at 01000000 ...
│   │   │    <>    Image Name:   Linux
│   │   │    <>    Image Type:   AArch64 Linux Kernel Image (lz4 compressed)
│   │   │    <>    Data Size:    7572857 Bytes = 7.2 MiB
│   │   │    <>    Load Address: 00400000
│   │   │    <>    Entry Point:  00400000
│   │   │    <>    Verifying Checksum ... OK
│   │   │    <> ## Loading init Ramdisk from Legacy Image at 09000000 ...
│   │   │    <>    Image Name:   Initrd
│   │   │    <>    Image Type:   AArch64 Linux RAMDisk Image (lz4 compressed)
│   │   │    <>    Data Size:    11968996 Bytes = 11.4 MiB
│   │   │    <>    Load Address: 00000000
│   │   │    <>    Entry Point:  00000000
│   │   │    <>    Verifying Checksum ... OK
│   │   │    <> ## Flattened Device Tree blob at 08000000
│   │   │    <>    Booting using the fdt blob at 0x8000000
│   │   │    <>    Uncompressing Kernel Image
│   │   │    <>    Loading Ramdisk to 3dc90000, end 3e7fa1e4 ... OK
│   │   │    <>    Loading Device Tree to 000000003dc89000, end 000000003dc8ff98 ... OK
│   │   │    <> 
│   │   │    <> Starting kernel ...
│   │   │    <> 
│   │   │    <> [    0.000000] Booting Linux on physical CPU 0x0000000000 [0x411fd040]
│   │   │    <> [    0.000000] Linux version 6.1.54-rt15-g899b22ebd57d (wangxiaoyong@AE-virtual-machine) (aarch64-linux-gnu-gcc (Linaro GCC 7.5-2019.12) 7.5.0, GNU ld (Linaro_Binutils-2019.12) 2.28.2.20170706) #1 SMP PREEMPT Mon Dec 18 16:40:10 CST 2023
│   │   │    <> [    0.000000] Machine model: Anlogic, DR1M90 FPSoc
│   │   │    <> [    0.000000] earlycon: uart0 at MMIO32 0x00000000f8401000 (options '')
│   │   │    <> [    0.000000] printk: bootconsole [uart0] enabled
│   │   │    <> [    0.000000] efi: UEFI not found.
│   │   │    <> [    0.000000] Reserved memory: created CMA memory pool at 0x000000002dc00000, size 256 MiB
│   │   │    <> [    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
│   │   │    <> [    0.000000] Zone ranges:
│   │   │    <> [    0.000000]   DMA      [mem 0x0000000000000000-0x000000003fffffff]
│   │   │    <> [    0.000000]   DMA32    empty
│   │   │    <> [    0.000000]   Normal   empty
│   │   │    <> [    0.000000] Movable zone start for each node
│   │   │    <> [    0.000000] Early memory node ranges
│   │   │    <> [    0.000000]   node   0: [mem 0x0000000000000000-0x000000003fffffff]
│   │   │    <> [    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x000000003fffffff]
│   │   │    <> [    0.000000] psci: probing for conduit method from DT.
│   │   │    <> [    0.000000] psci: PSCIv0.2 detected in firmware.
│   │   │    <> [    0.000000] psci: Using standard PSCI v0.2 function IDs
│   │   │    <> [    0.000000] psci: MIGRATE_INFO_TYPE not supported.
│   │   │    <> [    0.000000] percpu: Embedded 21 pages/cpu s45224 r8192 d32600 u86016
│   │   │    <> [    0.000000] pcpu-alloc: s45224 r8192 d32600 u86016 alloc=21*4096
│   │   │    <> [    0.000000] pcpu-alloc: [0] 0 [0] 1 
│   │   │    <> [    0.000000] Detected VIPT I-cache on CPU0
│   │   │    <> [    0.000000] CPU features: detected: GIC system register CPU interface
│   │   │    <> [    0.000000] alternatives: applying boot alternatives
│   │   │    <> [    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 258048
│   │   │    <> [    0.000000] Kernel command line: console=ttyS0,115200n8 earlycon=uart,mmio32,0xf8401000 loglevel=8 mtdparts=alspinor:1M(boot),-(data)
│   │   │    <> [    0.000000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
│   │   │    <> [    0.000000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
│   │   │    <> [    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
│   │   │    <> [    0.000000] Memory: 739080K/1048576K available (9024K kernel code, 1338K rwdata, 2792K rodata, 1472K init, 571K bss, 47352K reserved, 262144K cma-reserved)
│   │   │    <> [    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
│   │   │    <> [    0.000000] rcu: Preemptible hierarchical RCU implementation.
│   │   │    <> [    0.000000] rcu:   RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
│   │   │    <> [    0.000000]   Trampoline variant of Tasks RCU enabled.
│   │   │    <> [    0.000000]   Tracing variant of Tasks RCU enabled.
│   │   │    <> [    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
│   │   │    <> [    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
│   │   │    <> [    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
│   │   │    <> [    0.000000] GICv3: 160 SPIs implemented
│   │   │    <> [    0.000000] GICv3: 0 Extended SPIs implemented
│   │   │    <> [    0.000000] Root IRQ handler: gic_handle_irq
│   │   │    <> [    0.000000] GICv3: GICv3 features: 16 PPIs
│   │   │    <> [    0.000000] GICv3: CPU0: found redistributor 0 region 0:0x00000000dd040000
│   │   │    <> [    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
│   │   │    <> [    0.000000] arch_timer: cp15 timer(s) running at 33.33MHz (virt).
│   │   │    <> [    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x7b00c4bad, max_idle_ns: 440795202744 ns
│   │   │    <> [    0.000001] sched_clock: 56 bits at 33MHz, resolution 30ns, wraps every 2199023255551ns
│   │   │    <> [    0.008265] Console: colour dummy device 80x25
│   │   │    <> [    0.012741] Calibrating delay loop (skipped), value calculated using timer frequency.. 66.66 BogoMIPS (lpj=133333)
│   │   │    <> [    0.023068] pid_max: default: 32768 minimum: 301
│   │   │    <> [    0.027900] Mount-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
│   │   │    <> [    0.035292] Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
│   │   │    <> [    0.044663] cblist_init_generic: Setting adjustable number of callback queues.
│   │   │    <> [    0.051905] cblist_init_generic: Setting shift to 1 and lim to 1.
│   │   │    <> [    0.058069] cblist_init_generic: Setting adjustable number of callback queues.
│   │   │    <> [    0.065288] cblist_init_generic: Setting shift to 1 and lim to 1.
│   │   │    <> [    0.071540] rcu: Hierarchical SRCU implementation.
│   │   │    <> [    0.076338] rcu:   Max phase no-delay instances is 1000.
│   │   │    <> [    0.081897] EFI services will not be available.
│   │   │    <> [    0.086708] smp: Bringing up secondary CPUs ...
│   │   │    <> [    0.091805] Detected VIPT I-cache on CPU1
│   │   │    <> [    0.091922] GICv3: CPU1: found redistributor 1 region 0:0x00000000dd060000
│   │   │    <> 
│   │   │    <> [    0.091978] CPU1: Booted secondary processor 0x0000000001 [0x411fd040]
│   │   │    <> [    0.092111] smp: Brought up 1 node, 2 CPUs
│   │   │    <> [    0.113606] SMP: Total of 2 processors activated.
│   │   │    <> [    0.118307] CPU features: detected: 32-bit EL0 Support
│   │   │    <> [    0.123462] CPU features: detected: CRC32 instructions
│   │   │    <> [    0.128676] CPU: All CPU(s) started at EL1
│   │   │    <> [    0.132767] alternatives: applying system-wide alternatives
│   │   │    <> [    0.139636] devtmpfs: initialized
│   │   │    <> [    0.147839] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
│   │   │    <> [    0.157579] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
│   │   │    <> [    0.174421] DMI not present or invalid.
│   │   │    <> [    0.179140] NET: Registered PF_NETLINK/PF_ROUTE protocol family
│   │   │    <> [    0.186318] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
│   │   │    <> 
│   │   │    <> [    0.193481] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
│   │   │    <> [    0.201346] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
│   │   │    <> [    0.209322] audit: initializing netlink subsys (disabled)
│   │   │    <> [    0.214998] audit: type=2000 audit(0.140:1): state=initialized audit_enabled=0 res=1
│   │   │    <> [    0.215509] thermal_sys: Registered thermal governor 'fair_share'
│   │   │    <> [    0.222742] thermal_sys: Registered thermal governor 'bang_bang'
│   │   │    <> [    0.228820] thermal_sys: Registered thermal governor 'step_wise'
│   │   │    <> [    0.235128] ASID allocator initialised with 65536 entries
│   │   │    <> [    0.263897] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
│   │   │    <> [    0.270707] HugeTLB: 0 KiB vmemmap can be freed for a 1.00 GiB page
│   │   │    <> [    0.277009] HugeTLB: registered 32.0 MiB page size, pre-allocated 0 pages
│   │   │    <> [    0.283778] HugeTLB: 0 KiB vmemmap can be freed for a 32.0 MiB page
│   │   │    <> [    0.290030] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
│   │   │    <> [    0.296796] HugeTLB: 0 KiB vmemmap can be freed for a 2.00 MiB page
│   │   │    <> [    0.303049] HugeTLB: registered 64.0 KiB page size, pre-allocated 0 pages
│   │   │    <> [    0.309815] HugeTLB: 0 KiB vmemmap can be freed for a 64.0 KiB page
│   │   │    <> [    0.317239] dw_dmac f804d000.ahb_dma: WARN: Device release is not defined so it is not safe to unbind this driver while in use
│   │   │    <> [    0.329050] dw_dmac f804d000.ahb_dma: DesignWare DMA Controller, 8 channels
│   │   │    <> [    0.336754] SCSI subsystem initialized
│   │   │    <> [    0.340776] usbcore: registered new interface driver usbfs
│   │   │    <> [    0.346301] usbcore: registered new interface driver hub
│   │   │    <> [    0.351638] usbcore: registered new device driver usb
│   │   │    <> [    0.357465] pps_core: LinuxPPS API ver. 1 registered
│   │   │    <> [    0.362425] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
│   │   │    <> [    0.371543] PTP clock support registered
│   │   │    <> [    0.375866] FPGA manager framework
│   │   │    <> [    0.380254] vgaarb: loaded
│   │   │    <> [    0.383330] clocksource: Switched to clocksource arch_sys_counter
│   │   │    <> [    0.398926] NET: Registered PF_INET protocol family
│   │   │    <> [    0.404072] IP idents hash table entries: 16384 (order: 5, 131072 bytes, linear)
│   │   │    <> [    0.413617] tcp_listen_portaddr_hash hash table entries: 512 (order: 1, 8192 bytes, linear)
│   │   │    <> [    0.422054] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
│   │   │    <> [    0.429809] TCP established hash table entries: 8192 (order: 4, 65536 bytes, linear)
│   │   │    <> [    0.437650] TCP bind hash table entries: 8192 (order: 6, 262144 bytes, linear)
│   │   │    <> [    0.445163] TCP: Hash tables configured (established 8192 bind 8192)
│   │   │    <> [    0.451623] UDP hash table entries: 512 (order: 2, 16384 bytes, linear)
│   │   │    <> [    0.458269] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes, linear)
│   │   │    <> [    0.465552] NET: Registered PF_UNIX/PF_LOCAL protocol family
│   │   │    <> [    0.471740] RPC: Registered named UNIX socket transport module.
│   │   │    <> [    0.477662] RPC: Registered udp transport module.
│   │   │    <> [    0.482362] RPC: Registered tcp transport module.
│   │   │    <> [    0.487059] RPC: Registered tcp NFSv4.1 backchannel transport module.
│   │   │    <> [    0.493492] PCI: CLS 0 bytes, default 64
│   │   │    <> [    0.497873] Trying to unpack rootfs image as initramfs...
│   │   │    <> [    0.504051] workingset: timestamp_bits=62 max_order=18 bucket_order=0
│   │   │    <> [    0.524075] NFS: Registering the id_resolver key type
│   │   │    <> [    0.529343] Key type id_resolver registered
│   │   │    <> [    0.533654] Key type id_legacy registered
│   │   │    <> [    0.537769] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
│   │   │    <> [    0.544550] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
│   │   │    <> [    0.552014] jffs2: version 2.2. (NAND) © 2001-2006 Red Hat, Inc.
│   │   │    <> [    0.629955] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 244)
│   │   │    <> [    0.646456] dma-pl330 f8419000.dma-controller: WARN: Device release is not defined so it is not safe to unbind this driver while in use
│   │   │    <> [    0.659391] dma-pl330 f8419000.dma-controller: Loaded driver for PL330 DMAC-341330
│   │   │    <> [    0.667024] dma-pl330 f8419000.dma-controller:   DBUFF-16x16bytes Num_Chans-8 Num_Peri-4 Num_Events-16
│   │   │    <> [    0.680234] Serial: 8250/16550 driver, 2 ports, IRQ sharing disabled
│   │   │    <> [    0.688238] printk: console [ttyS0] disabled
│   │   │    <> [    0.712935] f8401000.serial: ttyS0 at MMIO 0xf8401000 (irq = 28, base_baud = 3125000) is a U6_16550A
│   │   │    <> [    0.722446] printk: console [ttyS0] enabled
│   │   │    <> [    0.722462] printk: bootconsole [uart0] disabled
│   │   │    <> [    0.723972] printk: console [ttyS0] printing thread started
│   │   │    <> [    0.739912] brd: module loaded
│   │   │    <> [    0.745708] Freeing initrd memory: 11688K
│   │   │    <> [    0.747630] loop: module loaded
│   │   │    <> [    0.748741] dw_spi_mmio f8404000.spi: Synopsys DWC APB SSI v4.03
│   │   │    <> [    0.748763] dw_spi_mmio f8404000.spi: Detected FIFO size: 32 bytes
│   │   │    <> 
│   │   │    <> [    0.748773] dw_spi_mmio f8404000.spi: Detected 32-bits max data frame size
│   │   │    <> [    0.748863] dw_dmac f804d000.ahb_dma: private_candidate: dma0chan0 filter said false
│   │   │    <> [    0.748877] dmaengine: __dma_request_channel: success (dma0chan1)
│   │   │    <> [    0.748906] dmaengine: __dma_request_channel: success (dma0chan0)
│   │   │    <> [    0.749106] dw_spi_mmio f8404000.spi: registered master spi0
│   │   │    <> [    0.749327] spi spi0.0: setup mode 0, 8 bits/w, 8000000 Hz max --> 0
│   │   │    <> [    0.749729] dw_spi_mmio f8404000.spi: registered child spi0.0
│   │   │    <> 
│   │   │    <> [    0.749968] dw_spi_mmio f8405000.spi: Synopsys DWC APB SSI v4.03
│   │   │    <> [    0.749987] dw_spi_mmio f8405000.spi: Detected FIFO size: 32 bytes
│   │   │    <> [    0.749997] dw_spi_mmio f8405000.spi: Detected 32-bits max data frame size
│   │   │    <> [    0.750058] dw_dmac f804d000.ahb_dma: private_candidate: dma0chan0 busy
│   │   │    <> [    0.750069] dw_dmac f804d000.ahb_dma: private_candidate: dma0chan1 busy
│   │   │    <> [    0.750079] dw_dmac f804d000.ahb_dma: private_candidate: dma0chan2 filter said false
│   │   │    <> [    0.750089] dmaengine: __dma_request_channel: success (dma0chan3)
│   │   │    <> [    0.750121] dw_dmac f804d000.ahb_dma: private_candidate: dma0chan0 busy
│   │   │    <> [    0.750130] dw_dmac f804d000.ahb_dma: private_candidate: dma0chan1 busy
│   │   │    <> [    0.750139] dmaengine: __dma_request_channel: success (dma0chan2)
│   │   │    <> [    0.750313] dw_spi_mmio f8405000.spi: registered master spi1
│   │   │    <> [    0.750478] spi spi1.0: setup mode 0, 8 bits/w, 8000000 Hz max --> 0
│   │   │    <> [    0.750912] dw_spi_mmio f8405000.spi: registered child spi1.0
│   │   │    <> [    0.751499] al-qspi f804e000.spi: registered master spi2
│   │   │    <> [    0.751664] spi spi2.0: setup mode 0, 8 bits/w, 10000000 Hz max --> 0
│   │   │    <> [    0.751926] spi-nor spi2.0: n25q128a11 (16384 Kbytes)
│   │   │    <> [    0.751997] 2 cmdlinepart partitions found on MTD device alspinor
│   │   │    <> [    0.752007] Creating 2 MTD partitions on "alspinor":
│   │   │    <> [    0.752017] 0x000000000000-0x000000100000 : "boot"
│   │   │    <> [    0.753028] 0x000000100000-0x000001000000 : "data"
│   │   │    <> [    0.753967] al-qspi f804e000.spi: registered child spi2.0
│   │   │    <> [    0.755091] CAN device driver interface
│   │   │    <> [    0.757292] dwc-eth-dwmac f8100000.ethernet0: User ID: 0x10, Synopsys ID: 0x52
│   │   │    <> [    0.757312] dwc-eth-dwmac f8100000.ethernet0:   DWMAC4/5
│   │   │    <> [    0.757323] dwc-eth-dwmac f8100000.ethernet0: DMA HW capability register supported
│   │   │    <> [    0.757332] dwc-eth-dwmac f8100000.ethernet0: RX Checksum Offload Engine supported
│   │   │    <> [    0.757341] dwc-eth-dwmac f8100000.ethernet0: TX Checksum insertion supported
│   │   │    <> [    0.757349] dwc-eth-dwmac f8100000.ethernet0: TSO supported
│   │   │    <> [    0.757357] dwc-eth-dwmac f8100000.ethernet0: Enable RX Mitigation via HW Watchdog Timer
│   │   │    <> 
│   │   │    <> [    0.757367] dwc-eth-dwmac f8100000.ethernet0: Enabled RFS Flow TC (entries=10)
│   │   │    <> [    0.757383] dwc-eth-dwmac f8100000.ethernet0: TSO feature enabled
│   │   │    <> [    0.757393] dwc-eth-dwmac f8100000.ethernet0: Using 32/32 bits DMA host/device width
│   │   │    <> [    0.766769] dwc2 f8180000.usb: supply vusb_d not found, using dummy regulator
│   │   │    <> [    0.766944] dwc2 f8180000.usb: supply vusb_a not found, using dummy regulator
│   │   │    <> [    0.767513] dwc2 f8180000.usb: DWC OTG Controller
│   │   │    <> [    0.767546] dwc2 f8180000.usb: new USB bus registered, assigned bus number 1
│   │   │    <> [    0.767578] dwc2 f8180000.usb: irq 35, io mem 0xf8180000
│   │   │    <> [    0.767794] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.01
│   │   │    <> [    0.767810] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
│   │   │    <> [    0.767822] usb usb1: Product: DWC OTG Controller
│   │   │    <> [    0.767832] usb usb1: Manufacturer: Linux 6.1.54-rt15-g899b22ebd57d dwc2_hsotg
│   │   │    <> [    0.767842] usb usb1: SerialNumber: f8180000.usb
│   │   │    <> [    0.768445] hub 1-0:1.0: USB hub found
│   │   │    <> [    0.768482] hub 1-0:1.0: 1 port detected
│   │   │    <> [    0.769671] usbcore: registered new interface driver uas
│   │   │    <> [    0.769760] usbcore: registered new interface driver usb-storage
│   │   │    <> [    0.770003] i2c_dev: i2c /dev entries driver
│   │   │    <> Starting syslogd: [    0.773024] i2c i2c-0: Added multiplexed i2c bus 1
│   │   │    <> OK
│   │   │    <> [    0.773222] i2c i2c-0: Added multiplexed i2c bus 2
│   │   │    <> [    0.773635] at24 3-0054: supply vcc not found, using dummy regulator
│   │   │    <> [    0.792732] at24 3-0054: 1024 byte 24c08 EEPROM, writable, 16 bytes/write
│   │   │    <> Starting klogd: [    0.792815] i2c i2c-0: Added multiplexed i2c bus 3
│   │   │    <> [    0.856635] rtc-ds1307 4-0068: registered as rtc0
│   │   │    <> OK
│   │   │    <> [    0.869699] rtc-ds1307 4-0068: setting system clock to 2000-01-01T20:24:10 UTC (946758250)
│   │   │    <> [    0.869785] i2c i2c-0: Added multiplexed i2c bus 4
│   │   │    <> [    0.869984] i2c i2c-0: Added multiplexed i2c bus 5
│   │   │    <> [    0.870176] i2c i2c-0: Added multiplexed i2c bus 6
│   │   │    <> [    0.870357] i2c i2c-0: Added multiplexed i2c bus 7
│   │   │    <> Running sysctl: [    0.870546] i2c i2c-0: Added multiplexed i2c bus 8
│   │   │    <> [    0.870561] pca954x 0-0074: registered 8 multiplexed busses for I2C switch pca9548
│   │   │    <> [    0.871085] sdhci: Secure Digital Host Controller Interface driver
│   │   │    <> [    0.871090] sdhci: Copyright(c) Pierre Ossman
│   │   │    <> [    0.871097] sdhci-pltfm: SDHCI platform and OF driver helper
│   │   │    <> OK[    0.872553] usbcore: registered new interface driver usbhid
│   │   │    <> 
│   │   │    <> [    0.872563] usbhid: USB HID core driver
│   │   │    <> [    0.873382] ipip: IPv4 and MPLS over IPv4 tunneling driver
│   │   │    <> [    0.873989] gre: GRE over IPv4 demultiplexor driver
│   │   │    <> Saving random seed: [    0.873995] ip_gre: GRE over IPv4 tunneling driver
│   │   │    <> [    0.875303] IPv4 over IPsec tunneling driver
│   │   │    <> [    0.876563] NET: Registered PF_INET6 protocol family
│   │   │    <> 
│   │   │    <> [    0.879120] Segment Routing with IPv6
│   │   │    <> [    0.879180] In-situ OAM (IOAM) with IPv6
│   │   │    <> [    0.880005] NET: Registered PF_PACKET protocol family
│   │   │    <> [    0.880026] NET: Registered PF_KEY protocol family
│   │   │    <> [    0.880032] can: controller area network core
│   │   │    <> [    0.880174] NET: Registered PF_CAN protocol family
│   │   │    <> [    0.880183] can: raw protocol
│   │   │    <> [    0.880192] can: broadcast manager protocol
│   │   │    <> [    0.880203] can: netlink gateway - max_hops=1
│   │   │    <> [    0.880600] Key type dns_resolver registered
│   │   │    <> [    0.889997] Key type encrypted registered
│   │   │    <> [    0.895053] of-fpga-region soc:base_fpga_region: FPGA Region probed
│   │   │    <> [    0.896449] input: soc:keys as /devices/platform/soc/soc:keys/input/input0
│   │   │    <> [    0.906383] of_cfs_init
│   │   │    <> [    0.906491] of_cfs_init: OK
│   │   │    <> [    0.907408] mmc0: SDHCI controller on f8049000.mmc [f8049000.mmc] using ADMA
│   │   │    <> [    0.908381] Freeing unused kernel memory: 1472K
│   │   │    <> [    0.967923] Checked W+X mappings: passed, no W+X pages found
│   │   │    <> [    0.967958] Run /init as init process
│   │   │    <> [    0.967966]   with arguments:
│   │   │    <> [    0.967968]     /init
│   │   │    <> [    0.967972]   with environment:
│   │   │    <> [    0.967975]     HOME=/
│   │   │    <> [    0.967978]     TERM=linux
│   │   │    <> [    1.058909] mmc0: new high speed SDHC card at address aaaa
│   │   │    <> [    1.063692] mmcblk0: mmc0:aaaa SL16G 14.8 GiB 
│   │   │    <> [    1.075850]  mmcblk0: p1
│   │   │    <> 
│   │   │    <> [    2.243349] random: crng init done
│   │   │    <> OK
│   │   │    <> Starting network: OK
│   │   │    <> ssh-keygen: generating new host keys: RSA DSA ECDSA ED25519 
│   │   │    <> Starting sshd: OK
│   │   │    <> Starting linuxptp daemon: OK
│   │   │    <> Starting linuxptp system clock synchronization: OK
│   │   │    <> 
│   │   │    <> 
│   │   │    <> Welcome to Buildroot
│   │   │    <> buildroot login: root
│   │   │    <> Password: 
│   │   ├─Set up the testing environment
│   │   ├─[board_dr1x90_ad101_v10_-linux] mkdir -p /boot
│   │   ├─[board_dr1x90_ad101_v10_-linux] mount /dev/mmcblk0p3 /data
│   │   │    ## mount: mounting /dev/mmcblk0p3 on /data failed: No such file or directory
│   │   ├─POWEROFF (board_dr1x90_ad101_v10)
│   │   ├─Power off
│   │   └─Fail. (17.780s)
FAILED
```

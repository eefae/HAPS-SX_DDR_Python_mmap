# HAPS-SX_DDR_Python_mmap
## ● 首先由CX5 SERVER電腦連入SMF卡(MPSOC卡)
* Last login: Mon Feb 26 14:28:45 2024 from 59.124.169.157
* haps@cx5:~$
* haps@cx5:~$ ssh xilinx@192.168.50.227 -X
The authenticity of host '192.168.50.227 (192.168.50.227)' can't be established.
ECDSA key fingerprint is SHA256:Sgff/AT5Bn/9XjlTfuJ4zMbNjDWeqY9ymoOTSy91uxc.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.50.227' (ECDSA) to the list of known hosts.
xilinx@192.168.50.227's password:
Welcome to PYNQ Linux, based on Ubuntu 22.04 (GNU/Linux 5.15.36-xilinx-v2022.2 aarch64)
Last login: Tue Feb 27 02:59:21 2024
xilinx@pynq:~$

## ● 查看SMF卡認到的DDR，其中0x01000000000起16G為外部HAPS主機的DDR
xilinx@pynq:~$ lsmem
RANGE                                 SIZE  STATE REMOVABLE   BLOCK
0x0000000000000000-0x000000007fffffff   2G online        no    0-15
0x0000000800000000-0x000000087fffffff   2G online        no 256-271
0x0000001000000000-0x00000013ffffffff  16G online        no 512-639
Memory block size:       128M
Total online memory:      20G
Total offline memory:      0B
xilinx@pynq:~$

## ●執行一BASH檔，其為使用devmem指令來填寫一些以0x01000000000為起始的資料
xilinx@pynq:~$ bash ~/nfs/driver/devmem-ddr.sh
[sudo] password for xilinx:
0xA5A5A5A5
0xB5B5B5B5
0xC5C5C5C5
0xD5D5D5D5
0xF5F5F5F5
1000000000  a5 a5 a5 a5 b5 b5 b5 b5  c5 c5 c5 c5 d5 d5 d5 d5  |................|
1000000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
1000000020  00 00 00 00 f5 f5 f5 f5  00 00 00 00 00 00 00 00  |................|
1000000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
1000000040  ff ff ff ff ff fd ff ff  ff ff ff ff ff ff ff ff  |................|
1000000050  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
1000000070  ff ff ff ff ff ff f7 ff  ff ff ff ff ff ff ff ff  |................|
1000000080  a9 aa ff ff ff ff ff ff  7f ff ff ff ff ff ff ff  |................|
1000000090  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
xilinx@pynq:~$

## ●使用devmem2指令把位址0x01000000080為起始的兩BYTE填為零，
再重複執行上面的BASH檔，確認是否已清為零。
xilinx@pynq:~$ sudo devmem2 0x1000000080 h 0x0000
/dev/mem opened.
Memory mapped at address 0xffffba08e000.
Value at address 0x80 (0xffffba08e080): 0xAAA9
Written 0x0; readback 0x0
xilinx@pynq:~$
xilinx@pynq:~$ bash ~/nfs/driver/devmem-ddr.sh
0xA5A5A5A5
0xB5B5B5B5
0xC5C5C5C5
0xD5D5D5D5
0xF5F5F5F5
1000000000  a5 a5 a5 a5 b5 b5 b5 b5  c5 c5 c5 c5 d5 d5 d5 d5  |................|
1000000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
1000000020  00 00 00 00 f5 f5 f5 f5  00 00 00 00 00 00 00 00  |................|
1000000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
1000000040  ff ff ff ff ff fd ff ff  ff ff ff ff ff ff ff ff  |................|
1000000050  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
1000000070  ff ff ff ff ff ff f7 ff  ff ff ff ff ff ff ff ff  |................|
1000000080  00 00 ff ff ff ff ff ff  7f ff ff ff ff ff ff ff  |................|
1000000090  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
xilinx@pynq:~$

## ● 使用者自行創立一PYTHON程式來任意讀寫物理記憶體
由於一般程式的資料位址是虛擬的位址而非物理位址，若要讀寫物理記憶體
必須引入mmap功能才行。
DDR RAM在LINUX系統中看成是一種裝置，也看成是一種檔案(稱 /dev/mem)。
程式中會指定offset = 0x01000000000。是指檔案(RAM)中的偏移。
程式中會開啟一塊128byte的虛擬記憶體(名mmap_obj[])與物理記憶體對應。
執行方法與結果：
xilinx@pynq:~$ sudo python3 ~/KC/mmap_demo02.py
0x00: 165              -----> 即 0xA5 , 表示能看到之前BASH程式的物理記憶體
0x01: 165              -----> 即 0xA5           
0x02: 165              -----> 即 0xA5
0x03: 165              -----> 即 0xA5
0x80: 0                 -----> 即在0x01000000080 的內容是0x00
change 0x80's to: 169         -----> 將位址0x01000000080的內容 由 0x00 改為 0xA5
xilinx@pynq:~$

## ● 再重複執行一次BASH檔，確認使用者的程式能改變到指定的物理記憶體。
xilinx@pynq:~$ bash ~/nfs/driver/devmem-ddr.sh
0xA5A5A5A5
0xB5B5B5B5
0xC5C5C5C5
0xD5D5D5D5
0xF5F5F5F5
1000000000  a5 a5 a5 a5 b5 b5 b5 b5  c5 c5 c5 c5 d5 d5 d5 d5  |................|
1000000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
1000000020  00 00 00 00 f5 f5 f5 f5  00 00 00 00 00 00 00 00  |................|
1000000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
1000000040  ff ff ff ff ff fd ff ff  ff ff ff ff ff ff ff ff  |................|
1000000050  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
1000000070  ff ff ff ff ff ff f7 ff  ff ff ff ff ff ff ff ff  |................|
1000000080  a9 aa ff ff ff ff ff ff  7f ff ff ff ff ff ff ff  |................|
1000000090  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
xilinx@pynq:~$

## ●附錄： mmap_demo02.py程式列印
import mmap
def mmap_io(filename):
    with open(filename, mode="r+", encoding="utf8") as file_obj:
                    with mmap.mmap(file_obj.fileno(), length=256, access=mmap.ACCESS_WRITE,offset=0x01000000000) as mmap_obj:
                                    print("0x00:",mmap_obj[0])
                                    print("0x01:",mmap_obj[1])
                                    print("0x02:",mmap_obj[2])
                                    print("0x03:",mmap_obj[3])
                                    print("0x80:",mmap_obj[128])
                                    mmap_obj[128]=169
                                    mmap_obj[129]=170
                                    print("change 0x80's to:",mmap_obj[128])
mmap_io("/dev/mem")



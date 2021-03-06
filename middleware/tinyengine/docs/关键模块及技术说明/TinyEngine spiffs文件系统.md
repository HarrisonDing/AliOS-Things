## 为什么需要spiffs文件系统
由于TinyEngine需要保存本地的js文件和板级配置。也需要在线更新这些js文件和板级配置，同时也支持本地实时调试和运行。所以我们考虑了使用key-value机制和文件系统机制两种方法来保存，考虑到key-value的扩展性不强，最终选择使用文件系统。由于TinyEngine主要面向嵌入式操作系统，使用的flash基本上是nor flash/ spi flash。
所以选择spiffs文件系统就比较适合了。spiffs文件系统专门针对使用NOR技术的SPI FLASH设计。采用了写平衡技术和GC垃圾回收技术，效率和稳定性较高。


## 各个硬件模组Flash空间对比

   
__不同的模组剩余的rom空间不一样，根据实际情况配置。__

| EMW3060  | Esp32devkitc | stm32（板子型号待定） |
| --- | --- | --- |
| Flash容量总计 2M | Flash容量总计4M | Flash容量总计1M |
| booloader :（64K） | bootloader       :（28K） |  |
| Parameter1 :（4K） | Parameter1        :（4K） | Parameter1        :（4K） |
| Parameter2 :   (4K)  | Parameter2       :（ 8K） | Parameter2        :（8K） |
| Application:  (568K)  | application      : （1M） | application      : （240K） |
| OTA         : (568K)  | ota        : （1M）  |  |
| Parameter3       : (4K)  | Paramter3 :（4K）   | Paramter3 :（4K）   |
| parameter4         : (4K) | Paramter4   :（4K）   |  |
|  剩余给spiffs 共：__  832K __ | <span data-type="color" style="color:#262626">剩余给spiffs共：  </span><strong><span data-type="color" style="color:#262626">2000K  </span></strong><strong><span data-type="color" style="color:#FA8C16">  </span></strong><strong> </strong> |  |

## gravity@lite spiffs文件系统分区配置信息
__<span data-type="color" style="color:#FA541C">注意：当前gravity@lite项目给spiffs文件系统划分的分区统一为512K。</span>__

| EMW3060 | Esp32devkitc | stm32（板子型号待定） |
| --- | --- | --- |
| 大小：256K byte | 大小：512K byte | 大小：256K byte |
| 起始地址 ： 0x185200 | 起始地址： 0x315000     | 起始地址：0x80C0000 |
| block大小：64K | block大小：64K | block大小：16K |
| page大小： 256 byte | page大小： 256 byte | page大小： 2K |


## 本地文件打包成spiffs镜像方法
* 在gravity@lite的编译系统中，执行编译命令__ make__ 后，会自动打包 __/app/js__ 文件夹为一个spiffs文件系统的镜像，并自动生成到 build/output/spiffs.bin  。最后，您可以将这个spiffs.bin镜像通过bootloader模式的命令write 烧录到硬件模组到flash中。
      原理：事实上，make命令在编译app目录时，会自动调用命令mkspiffs，对应的makefile源码如下：
```bash
ifeq ($(PRODUCT_TYPE), developerkit)
IMAGE_SIZE=262144
BLOCK_SIZE=16384
PAGE_SIZE=2048
else ifeq ($(PRODUCT_TYPE), mk3060)
IMAGE_SIZE=262144
BLOCK_SIZE=65536
PAGE_SIZE=256
else
IMAGE_SIZE=524288
BLOCK_SIZE=65536
PAGE_SIZE=256
endif
$(MKSPIFFS) -c js/  -b $(BLOCK_SIZE) -p $(PAGE_SIZE) -s $(IMAGE_SIZE) spiffs.bin
```

## 烧写​spiffs镜像或读取spiffs分区的方法
1. EMW3060 烧写spiffs镜像方法
```c
1）连接emw3060的串口到PC 或者 按住BOOT键并连接usb到 PC。
2）在PC上打开SecureCRT，波特率921600。进入bootloader模式。
3）bootloader模式下输入：write 0x185200 ，提示使用secureCRT的SendYmodem选择你的spiffs.image
4）传输完成后，就将您的spiffs.bin镜像烧写到flash对应的地址了，mcu启动后，可以通过mount读取打包的文件。
```

2. EMW3060 读取spiffs分区方法
```c
1）连接emw3060的串口到PC 或者 按住BOOT键并连接usb到 PC。
2）在PC上打开SecureCRT，波特率921600。进入bootloader模式。
3）bootloader模式下输入：read 0x185200 0x80000 命令去读取该地址分区内容，并选择使用Receive Ymodem方式接收。
默认接受后的文件名为image.bin。并保持在 “文档“ 目录下。
4）可以通过build/output目录下的mkspiffs解压这个image.bin镜像，解压生成一个文件目录。
命令如下：./mkspiffs -u fs/ -b 65536 -p 256 -s 262144 image.bin
```

3. ESP32 烧写spiffs分区方法：
__说明：__mac上端口一般为/dev/tty.SLAB\_USBtoUART，Linux系统上为/dev/ttyUSB0,请根据系统自行更改端口。
```plain
esptool.py --chip esp32 --port /dev/ttyUSB0 write_flash -z -p 0x315000 ./build/output/spiffs.bin
```
* 在头文件 be\_osal\_fs.h 中封装了这些接口为be\_osal\_, 可以通过include <be\_osal\_fs.h>使用它们。
* 默认spiffs分区mount加载后的目录名为/spiffs，所以，所有的文件路径都需要以/spiffs开头。

```c
 int be_osal_spiffs_init(void);
 int be_osal_spiffs_cleanup(void);
 int be_osal_open(const char *path, int flags);
 int be_osal_close(int fd);
 ssize_t be_osal_read(int fd, void *buf, size_t nbytes);
 ssize_t be_osal_write(int fd, const void *buf, size_t nbytes);
 int be_osal_sync(int fd);
 be_osal_dir_t *be_osal_opendir(const char *path);
 be_osal_dirent_t *be_osal_readdir(be_osal_dir_t *dir);
 int be_osal_closedir(be_osal_dir_t *dir);
 int be_osal_rename(const char *oldpath, const char *newpath);
 int be_osal_unlink(const char *path);
 int be_osal_stat(const char *path, struct stat *st);
```
* 使用示例
```c
int gravity_spiffs_init(void)
{ 	
	int ret = -1;

	be_debug(MAIN_TAG,"%s,%d\n\r",__FUNCTION__,__LINE__);
	ret = be_osal_spiffs_init();
	if (ret != 0){
		be_debug("be_osal_spiffs_init fail %d\n\r",ret);
		return -1;
	}
	be_debug(MAIN_TAG,"gravity_spiffs_init ok\n\r");
	be_spiffs_interface_test();//test open read write
}
```
  
## spiffs文件系统简介
[https://blog.csdn.net/zhangjinxing\_2006/article/details/75050611](https://blog.csdn.net/zhangjinxing_2006/article/details/75050611)

## spiffs移植FAQ
1. 在mcu上通过aos接口创建的文件，目录。通过read命令读取出来的spiffs分区镜像在pc上解压后文件内容不可读，为乱码。
 
原因：由于emw3060的编译器优化原因，在spiffs分区header的读取时，优化了内存奇地址的对齐方式，即在奇地址的16bit读写会强制找到最近的偶地址。而pc上的编译器并没有对此进行优化，所以导致文件地址读取出错，解析不出来。
解决方法：强制二字节对齐。 #define SPIFFS\_ALIGNED\_OBJECT\_INDEX\_TABLES       1

2. pc上打包的镜像烧录到flash上读取不出来，目录解析错误。
这是因为aos移植时，在解析文件的绝对路径自动去掉了/spiffs/test.txt中的/spiffs/ 这个头，导致相对路径为test.txt,而实际上pc打包的路径为/test.txt。所以修改了aos移植的文件路径解析方法。
详情查看这个函数：
```c
static char* translate_relative_path(const char *path)
```

 
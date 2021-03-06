# 功能特性

* 应用执行
    * 从本地FLASH加载并执行JS应用程序
    * 执行JS语法块，用于REPL-CLI
* 应用版本管理
    * 版本信息存储及上报        
* 应用升级
    * 从网络下载JS程序包`APP_PACK`
    * 校验、解密、保存 JS 程序

# 应用云端下发流程



![image.png | left | 748x573](https://cdn.yuque.com/lark/0/2018/png/8336/1524132443391-cd68c51a-4e1f-4b75-aaea-450ed36e8f20.png "")


__<span data-type="color" style="color:#FF4D4F">JS应用升级的风险：</span>__
有一个特殊标示， 标记当前app升级未正常结束的情况，该情况下启动再次更新应用程序，前提是网络正常。

从ESP32模组的分区表看，应用程序是有主备两个同等大小的分区，每次升级都是写备份分区，升级成功之后从备份分区启动并会变成主分区。
详细设计参考[AOS FOTA升级介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-FOTA-Porting-Guide?spm=a2c4e.11153959.blogcont309562.8.7c9327daZEOP4v)
```plain
	[HAL_PARTITION_APPLICATION] =
	{
	    .partition_owner            = HAL_FLASH_EMBEDDED,
	    .partition_description      = "Application",
	    .partition_start_addr       = 0x10000,
	    .partition_length           = 0x100000, //1MB bytes
	    .partition_options          = PAR_OPT_READ_EN | PAR_OPT_WRITE_EN,
	},
    [HAL_PARTITION_OTA_TEMP] =
    {
        .partition_owner           = HAL_FLASH_EMBEDDED,
        .partition_description     = "OTA Storage",
        .partition_start_addr      = 0x110000,
        .partition_length          = 0x100000, //1MB bytes
        .partition_options         = PAR_OPT_READ_EN | PAR_OPT_WRITE_EN,
    },
```

<span data-type="color" style="color:#FF4D4F"><strong>疑问：</strong></span>
1. （ProductKey/DeviceName/DeviceSecret）出厂已经烧录到设备参数分区中，这三要素升级FOTA固件时也需要，升级JS APP时也需要，通过系统API获取
2. FOTA升级固件 与 JS APP， 是一个通道，只是topic不同，需要与AOS团队一起商定
3. FOTA当前只能升级一个分区，在当前场景应用中需要升级多个分区，如参数分区、JS应用分区,
    需要与AOS团队一起商定
4. 当前有PARAMETER 1~4 ，4个分区的用途是什么？
5. 增加一个“JS\_APP”的分区，用于保存JS应用程序，剩余空间都分给该分区，建议不小于128KB

# app\_pack文件

APP\_PACK文件是对JS应用程序包
JS程序加密算法： AES-ECB-128  
内容完整性校验算法   ： MD5

## JS应用程序
```bash
.
├── drivers
│   ├── door.js
│   ├── fan.js
│   ├── light.js
│   └── temp.js
├── index.js
├── device.js
├── README.md
└── version.js
```

__应用程序说明：__
* __index.js__
JS应用程序执行入口
* __device.js__
设备上云相关操作类
* __version.js__
JS应用程序版本信息模块文件
* __drivers__
JS应用需要使用到的设备对象模块，与具体硬件相关，操作buildin的hw对象


### APP\___PACK文件说明：__

文件头结构：
```c
typedef struct {
    uint16 file_count;
    uint16 pack_version;
    uint32 pack_size;
} JSEPACK_HEADER;

typedef struct {
    uint8  name[32];
    uint32 file_type:8;
    uint32 file_size:24;
    uint8  md5[16];
    // uint8 data[];
}JSEPACK_FILE;

typedef struct {
    JSEPACK_HEADER file_header;
    //JSEPACK_FILE   file_bodys[];
}JSEPACK;
```

__<span data-type="color" style="color:#F5222D">文件名称长度约定：  少于32Byte</span>__

说明：
* __file\_header__
`JSEPACK_HEADER`类型，描述pack中文件的个数，及应用软件包版本，该文件的总长度等。
* __file\_bodys[]__
pack中包含的文件，每个文件都是`JSEPACK_FILE`类型，头信息有文件名称，类型，大小，从第52Byte开始是文件内容，文件内容采用`AES-ECB-128`算法加密，密钥采用云端下发
对文件的内容采用`MD5`算法计算哈希值并保存在pack文件头信息`file_header`中

### APP\___PACK打包工具__

__git库地址:__
```plain
git@gitlab.alibaba-inc.com:Gravity/jsepack.git
```

__使用方法:__
```bash
node app_pack  ./app
```
把当前`./app`目录对应的[JS应用程序](#JS应用程序)打包成`./app_pack.bin`文件


# API 接口

## int apppack\_upgrade(char\* url);
从指定URL下载app pack并升级，升级成功返回0




# 验证方法

1. 制作app pack
```bash
node app_pack  ./app
```
2. 搭建web server
```bash
npm install -g http-server
http-server  ./
```
3. 升级验证
```bash
在串口CLI中
appupdate http://192.168.8.101:8080/app_pack.bin
```

 
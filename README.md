# esp-idf-extended-decoders
Some extended decoders for esp-idf on Bluedroid.

目的是在esp32上验证A2DP拓展编码，目前AAC、aptX[-LL & -HD]、LDAC、LC3 Plus、OPUS已验证可用，LHDCV5已从AOSP移植过来了，但是不可用。

本仓库只处理esp-idf编码之间的适配问题，而关于编码方面的问题在仓库[extended-a2dp-bluedroid](https://github.com/sprlightning/extended-a2dp-bluedroid)中进行讨论，这里尽量不重复；

## android_external_lhdc

由mvaisakh创建，提供了"lhdcv5BT_dec.h"和"lhdcv5BT_dec.c"等内容，适用于Android，详见：[android_external_lhdc](https://github.com/mvaisakh/android_external_lhdc)。

关于lhdcv5的内容是android_external_lhdc/liblhdcv5/inc的lhdcv5BT.h；android_external_lhdc/liblhdcv5/include的lhdcv5BT_ext_func.h和lhdcv5_api.h；android_external_lhdc/liblhdcv5/src的lhdcv5BT_enc.c；

关于lhdcv5dev的内容是：android_external_lhdc/liblhdcv5dec/inc的lhdcv5BT_dec.h；android_external_lhdc/liblhdcv5dec/include的lhdcv5_util_dec.h；android_external_lhdc/liblhdcv5dec/src的lhdcv5BT_dec.c；

此外还有liblhdc和liblhdcdec，推测是小于V5的版本，但是在这里保留；

## android_packages_modules_Bluetooth

由web1n创建，提供了lhdc在内的多个编码的完整协议，包括编解码器，适用于Android，相对标准化，详见：[android_packages_modules_Bluetooth](https://github.com/web1n/android_packages_modules_Bluetooth)。

很明显这个包集成了绝大多数A2DP编码，其源文件位于android_packages_modules_Bluetooth/system/stack/a2dp目录，包括LHDCV5和LHDC其他版本以及大多数主流编码的编码器和解码器的源文件；目录android_packages_modules_Bluetooth/system/stack/include则是对应这些编解码器的头文件。

## esp-idf修改版

本仓库的目录esp-idf是esp-idf修改版的蓝牙部分，由cfint修改版（详见[esp-idf](https://github.com/cfint/esp-idf/tree/v5.1.4-a2dp-codecs)）再次修改而来，除了支持基本的SBC，还集成了AAC、aptX[-LL & -HD]、LDAC、LC3 Plus、OPUS编码，都已验证可用，此外还集成了LHDCV5编码，可能存在一些问题，尚不可用；

### 内容举例

下面以功能正常的LDAC为例来说明：

esp-idf/components/bt/host/bluedroid/stack/a2dp目录：是包括LDAC在内的各种拓展编码的源文件；

esp-idf/components/bt/host/bluedroid/stack/include/stack目录：是包括LDAC在内的各种拓展编码的头文件；

esp-idf/components/bt/host/bluedroid/external目录：是包括LDAC在内的各种拓展编码的external库；

相较于原生esp-idf，cfint修改版esp-idf的具体改动点如下(以LDAC为例)：

- esp-idf/components/bt目录：CMakeLists.txt：  
	增加了各种编码的条件编译，如“if(CONFIG_BT_A2DP_LDAC_DECODER) ... endif()”，目的是当条件满足时包含对应的头文件目录和对应的源文件；注意该目录下的Kconfig未改变，和原生IDF内容一致；

- esp-idf/components/bt/host/bluedroid目录：  
	Kconfig.in：开启了各种编码的menuconfig选项，如“config BT_A2DP_LDAC_DECODER ...”，可在menuconfig中控制是否启用该编码；

- esp-idf/components/bt/host/bluedroid/stack/a2dp目录：  
	a2dp_vendor.c：是一个内部的综合性的文件，包含"stack/a2dp_vendor_ldac.h"在内的各个解码器的头文件；由宏定义LDAC_DEC_INCLUDED是否为TRUE控制厂商ID、编码ID是否采用LDAC的值，以及返回对应的CIE，其余编码也有类似的操作；
	a2dp_vendor.c每种编码都包括10个函数，以ldac为例这10个函数定义a2dp_vendor_ldac.c，此外于还有更多内容，无法一一提及，需要阅读整个文件；
	
	a2dp_vendor_ldac.c：是与esp-idf内部的a2dp_vendor.c进行交互的中间文件，整个文件由宏定义LDAC_DEC_INCLUDED是否为TRUE控制内容是否启用，包括a2dp_vendor.c所需的10个函数的定义；
	包括A2DP_ParseInfoLdac、A2DP_CodecInfoMatchesCapabilityLdac等，功能详见a2dp_vendor_ldac.h的声明，无法一一提及，需要阅读整个文件；
	
	a2dp_vendor_ldac_decoder.c：ldac解码器相关，里面定义的函数会被a2dp_vendor_ldac.c调用；
	a2dp_vendor_ldac_decoder.c这个文件引入了ldacdec.h这个external库，而external库则是处理ldac编码的核心文件库，涉及具体的ldac算法；
	整个文件由宏定义LDAC_DEC_INCLUDED是否为TRUE控制内容是否启用；启用的内容包括tA2DP_LDAC_DECODER_CB结构体的定义和声明、ldac解码器初始化函数、packet_header函数、寻找同步word函数、packet函数、配置函数；

	a2dp_vendor_ldacbt_decoder.c：也是ldac解码器相关，这个文件也引入了ldacBT.h这个external库；
	整个文件由宏定义LDAC_DEC_INCLUDED是否为TRUE控制内容是否启用；

- esp-idf/components/bt/host/bluedroid/stack/a2dp/include目录：  
	bt_av.h：由宏定义LDAC_DEC_INCLUDED控制BTAV_A2DP_CODEC_INDEX_SOURCE_LDAC和BTAV_A2DP_CODEC_INDEX_SINK_LDAC的定义；

- esp-idf/components/bt/host/bluedroid/stack/include/stack目录：  
	a2dp_vendor_ldac.h：ldac解码器相关中间文件，此文件涉及至少10个ldac函数声明，会被a2dp_vendor_ldac.c直接调用，不一一列举，详见文件本身；
	
	a2dp_vendor_ldac_decoder.h：ldac解码器相关头文件，是解码器相关函数的声明，被a2dp_vendor_ldac.c调用，不一一列举，详见文件本身；
	
	a2dp_vendor_ldac_constants.h：各种ldac相关的宏定义，不一一列举，详见文件本身；

- esp-idf/components/bt/host/bluedroid/external目录：  
	是包括LDAC在内的各种拓展编码的external外部库；
	ldac相关内容就在esp-idf/components/bt/host/bluedroid/external/libldac-dec目录，涉及ldac核心算法和编码的具体实现逻辑，其中的ldacdec.h是对外的接口文件，由a2dp_vendor_ldac_decoder.c调用；
	a2dp_vendor_ldac_decoder.c再由a2dp_vendor_ldac.c调用；
	a2dp_vendor_ldac.c再由a2dp_vendor.c调用，分层调用实现ldac的解码；

- esp-idf/components/bt/host/bluedroid/api/include/api目录：  
	esp_a2dp_api.h：定义了esp_a2d_mcc_t结构体（A2DP media codec capabilities union），包括LDAC在内的各种编码的cie联合体；
	用途是临时存储配置参数如厂商ID、编码ID、采样率、位深、通道等内容，多种编码共用同一地址，每次只能存储一种编码，可以告知本设备支持哪些编码器，也可据此判断当前是什么编码;

- esp-idf/components/bt/host/bluedroid/btc/profile/std/a2dp/include目录：  
	bt_av_co.h：当BT_AV_INCLUDE为TRUE时，增加了对LDAC_DEC_INCLUDE是否定义为TRUE的判断，然后会据此控制启用一些BTC_SV_AV_AA_LDAC_INDEX或者BTC_SV_AV_AA_LDAC_SINK_INDEX枚举，涉及多个编码；

- esp-idf/components/bt/host/bluedroid/common/include/common目录：  
	bluedroid_user_config.h：增加了条件判断，如“#ifdef CONFIG_BT_A2DP_LDAC_DECODER #define UC_BT_A2DP_LDAC_DECODER_ENABLED    CONFIG_BT_A2DP_LDAC_DECODER #define UC_BT_A2DP_LDAC_DECODER_ENABLED    FALSE #endif”，用来控制相关的解码器是否启用；

- esp-idf/components/bt/host/bluedroid/common/include/common目录：  
	bt_target.h：当经典蓝牙启用时，判断对应的解码器是否有启用的宏定义（bluedroid_user_config.h中的宏定义），然后据此来定义LDAC_DEC_INCLUDE是否为TRUE；
	然后还有依据CONFIG_BT_A2DP_LDAC_DECODER是否定义来定义AVDT_LDAC_SEPS的值，进而定义AVDT_NUM_SEPS的值；

### LHDCV5的移植

#### 与esp-idf-Bluedroid相关的内容

我已经从AOSP（android_packages_modules_Bluetooth）移植与esp-idf-Bluedroid相关的内容，包括下列6个文件，并**已更新到本仓库**，均编译通过：
```c
extended-a2dp-bluedroid/250917_migrated_files/Bluedroid-a2dp-srcs：
- a2dp_vendor_lhdcv5.c
- a2dp_vendor_lhdcv5_decoder.c

extended-a2dp-bluedroid/250917_migrated_files/Bluedroid-a2dp-incs：
- a2dp_vendor_lhdcv5.h
- a2dp_vendor_lhdcv5_decoder.h
- a2dp_vendor_lhdc_constants.h
- a2dp_vendor_lhdcv5_constants.h
```

其中的a2dp_vendor_lhdc_constants.h和a2dp_vendor_lhdcv5_constants.h是直接从AOSP包里复制过来的，两个都有各自的用途；

#### lhdcv5解码器相关的外部库

我已经从android_external_lhdc移植了与lhdcv5解码器相关的外部库，内容如下，**已更新到本仓库**，均编译通过：
```c
extended-a2dp-bluedroid/250917_migrated_files/external-lib/liblhdcv5dec：
|- inc：
|-- lhdcv5BT_dec.h
|- include：
|-- lhdcv5_util_dec.h
|- src：
|-- lhdcv5_util_dec.c
|-- lhdcv5BT_dec.c
|- CMakeLists.txt
```
移植只涉及log方面的函数实现以及lhdcv5_util_dec.c的补充，所以函数结构能得到保障；

其中的lhdcv5_util_dec.c是依据lhdcv5_util_dec.h、lhdcv5BT_dec.c/.h这三个文件逆推出来的，**只有大致的框架，能做到编译通过，不具备实际解码功能**；

因为Savitech LHDC没有开源LHDC编码具体内容，大家通常是用的动态库，如lhdcv5_util_dec.so，不过我没法找到lhdcv5_util_dec.so这个文件；

#### esp-idf现存的文件的修改适配

对于esp-idf现存的文件的修改适配，具体如下，也已完成：  

- **修改 components/bt/CMakeLists.txt：**  
添加 lhdcv5 相关文件编译，风格尽量与其余编码保持一致，LHDCV5部分如下（这里是使用了external库liblhdcv5dec的情况）：
	```c
	if(CONFIG_BT_A2DP_LHDCV5_DECODER)
		list(APPEND priv_include_dirs host/bluedroid/external/liblhdcv5dec/inc
									  host/bluedroid/external/liblhdcv5dec/include)
		list(APPEND lhdcv5bt_dec_srcs "host/bluedroid/external/liblhdcv5dec/src/lhdcv5BT_dec.c"
									"host/bluedroid/external/liblhdcv5dec/src/lhdcv5_util_dec.c")
		list(APPEND srcs ${lhdcv5bt_dec_srcs})

		list(APPEND srcs "host/bluedroid/stack/a2dp/a2dp_vendor_lhdcv5.c"
						 "host/bluedroid/stack/a2dp/a2dp_vendor_lhdcv5_decoder.c")
	endif()
	```

- **修改 components/bt/host/bluedroid/Kconfig.in：**  
	添加 menuconfig 选项，风格也是尽量与其余编码保持一致，便于识别；
	LHDCV5选项是：
	```c
	config BT_A2DP_LHDCV5_DECODER
		bool "LHDCV5 decoder"
		depends on BT_A2DP_ENABLE
		default n
		help
			A2DP LHDCV5 decoder
	```

- **修改 components/bt/host/bluedroid/stack/a2dp/a2dp_vendor.c：**  
	添加 lhdcv5 支持，添加一个头文件“a2dp_vendor_lhdcv5.h”，和用宏定义包起来的十对有特定名称和功能的函数，风格自然也是与其余编码保持一致；
	对应的LHDCV5内容是：
	```c
	#if (defined(LHDCV5_DEC_INCLUDED) && LHDCV5_DEC_INCLUDED == TRUE)
	  // Check for LHDCV5
	  if (vendor_id == A2DP_LHDC_VENDOR_ID &&
		  codec_id == A2DP_LHDCV5_CODEC_ID) {
		return A2DP_ParseInfoLhdcv5((tA2DP_LHDCV5_CIE*)p_ie, p_codec_info, is_capability);
	  }
	#endif /* defined(LHDCV5_DEC_INCLUDED) && LHDCV5_DEC_INCLUDED == TRUE) */
	```

- **修改 components/bt/host/bluedroid/stack/a2dp/include/bt_av.h：**  
	添加 lhdcv5 编码索引，都是用宏定义包含的，具体是：BTAV_A2DP_CODEC_INDEX_SOURCE_LHDCV5和BTAV_A2DP_CODEC_INDEX_SINK_LHDCV5；
	LHDCV5是下面两个：
	```c
	#if (defined(LHDCV5_DEC_INCLUDED) && LHDCV5_DEC_INCLUDED == TRUE)
	  BTAV_A2DP_CODEC_INDEX_SOURCE_LHDCV5,
	#endif /* LHDCV5_DEC_INCLUDED */
	
	#if (defined(LHDCV5_DEC_INCLUDED) && LHDCV5_DEC_INCLUDED == TRUE)
	  BTAV_A2DP_CODEC_INDEX_SINK_LHDCV5,
	#endif /* LHDCV5_DEC_INCLUDED */
	```

- **修改 components/bt/host/bluedroid/common/include/common/bluedroid_user_config.h：**  
	增加宏定义配置，这个务必要统一命名风格，如下：
	```c
	#ifdef CONFIG_BT_A2DP_LHDCV5_DECODER
	#define UC_BT_A2DP_LHDCV5_DECODER_ENABLED    CONFIG_BT_A2DP_LHDCV5_DECODER
	#else
	#define UC_BT_A2DP_LHDCV5_DECODER_ENABLED    FALSE
	#endif
	```

- **修改 components/bt/host/bluedroid/common/include/common/bt_target.h：**
	增加宏定义配置，并配置AVDT_NUM_SEPS； 
	```c
	#if (UC_BT_A2DP_LHDCV5_DECODER_ENABLED == TRUE)
	#define LHDCV5_DEC_INCLUDED           TRUE
	#endif /* (UC_BT_A2DP_LHDCV5_DECODER_ENABLED == TRUE) */
	```

- **修改 components/bt/host/bluedroid/api/include/api/esp_a2dp_api.h：**
	添加 LHDCV5 到联合体，这个需要修改一下结构体__attribute__((packed)) esp_a2d_mcc_t；
	查询Android包里的a2dp_vendor_lhdc_constants.h可知A2DP_LHDCV5_CODEC_LEN是13，那么ESP_A2D_CIE_LEN_LHDCV5 = A2DP_LHDCV5_CODEC_LEN - 2 = 11；
	所以这个结构体可修改为：
	```c
	/**
	 * @brief A2DP media codec capabilities union
	 * 大部分CIE_LEN = CODEC_LEN - 2
	 */
	typedef struct {
		esp_a2d_mct_t type;                        /*!< A2DP media codec type */
	#define ESP_A2D_CIE_LEN_SBC          (4)
	#define ESP_A2D_CIE_LEN_M12          (4)
	#define ESP_A2D_CIE_LEN_M24          (10)
	#define ESP_A2D_CIE_LEN_ATRAC        (7)
	#define ESP_A2D_CIE_LEN_APTX         (7)
	#define ESP_A2D_CIE_LEN_APTX_HD      (11)
	#define ESP_A2D_CIE_LEN_APTX_LL      (7)
	#define ESP_A2D_CIE_LEN_LDAC         (8)
	#define ESP_A2D_CIE_LEN_OPUS         (26)
	#define ESP_A2D_CIE_LEN_LC3PLUS      (12)
	#define ESP_A2D_CIE_LEN_LHDCV5       (11)
		union {
			uint8_t sbc[ESP_A2D_CIE_LEN_SBC];      /*!< SBC codec capabilities */
			uint8_t m12[ESP_A2D_CIE_LEN_M12];      /*!< MPEG-1,2 audio codec capabilities */
			uint8_t m24[ESP_A2D_CIE_LEN_M24];      /*!< MPEG-2, 4 AAC audio codec capabilities */
			uint8_t atrac[ESP_A2D_CIE_LEN_ATRAC];  /*!< ATRAC family codec capabilities */
			uint8_t aptx[ESP_A2D_CIE_LEN_APTX];    /*!< APTX codec capabilities */
			uint8_t aptx_hd[ESP_A2D_CIE_LEN_APTX_HD];    /*!< APTX-HD codec capabilities */
			uint8_t aptx_ll[ESP_A2D_CIE_LEN_APTX_LL];    /*!< APTX-LL codec capabilities */
			uint8_t ldac[ESP_A2D_CIE_LEN_LDAC];    /*!< LDAC codec capabilities */
			uint8_t opus[ESP_A2D_CIE_LEN_OPUS];    /*!< OPUS codec capabilities */
			uint8_t lc3plus[ESP_A2D_CIE_LEN_LC3PLUS];    /*!< LC3 Plus codec capabilities */
			uint8_t lhdcv5[ESP_A2D_CIE_LEN_LHDCV5];    /*!< LHDCV5 codec capabilities */
		} cie;                                     /*!< A2DP codec information element */
	} __attribute__((packed)) esp_a2d_mcc_t;
	```

上面便是移植与适配LHDCV5修改的共用esp-idf的内容；

## 移植后的测试

抛开LHDC V5编码核心文件的问题，现在有个新的问题，就是蓝牙连接后会自动断开，esp-idf终端日志如下：
```c
W (2051271) BT_APPL: new conn_srvc id:19, app_id:0
I (2051741) WeiLeng-Player: bt连接状态: 2, 已连接
W (2054471) BT_HCI: hci cmd send: sniff: hdl 0x81, intv(400 800)
W (2054531) BT_HCI: hcif mode change: hdl 0x81, mode 2, intv 768, status 0x0
W (2064391) BT_HCI: hcif mode change: hdl 0x81, mode 0, intv 0, status 0x0
W (2064401) BT_APPL: bta_dm_act no entry for connected service cbs
W (2064401) BT_AVCT: avct_lcb_last_ccb
W (2064401) BT_AVCT: 0: aloc:1, lcb:0x0/0x3ffcf7fc, ccb:0x3ffcf864/0x3ffcf87c
W (2064411) BT_AVCT: 1: aloc:1, lcb:0x3ffcf7fc/0x3ffcf7fc, ccb:0x3ffcf87c/0x3ffcf87c
W (2064421) BT_AVCT: 2: aloc:0, lcb:0x0/0x3ffcf7fc, ccb:0x3ffcf894/0x3ffcf87c
I (2064431) WeiLeng-Player: bt连接状态: 0, 已断开
W (2068421) BT_HCI: hci cmd send: disconnect: hdl 0x81, rsn:0x13
W (2068511) BT_HCI: hcif disc complete: hdl 0x81, rsn 0x16
```
reset & retry：
```c
W (4581) BT_APPL: new conn_srvc id:19, app_id:0
I (5161) WeiLeng-Player: bt连接状态: 2, 已连接
W (7781) BT_HCI: hci cmd send: sniff: hdl 0x81, intv(400 800)
W (7781) BT_HCI: hcif mode change: hdl 0x81, mode 2, intv 768, status 0x0
W (20041) BT_HCI: hci cmd send: unsniff: hdl 0x81
W (20051) BT_APPL: bta_dm_act no entry for connected service cbs
W (20051) BT_AVCT: avct_lcb_last_ccb
W (20051) BT_AVCT: 0: aloc:1, lcb:0x0/0x3ffcf7fc, ccb:0x3ffcf864/0x3ffcf87c
W (20061) BT_AVCT: 1: aloc:1, lcb:0x3ffcf7fc/0x3ffcf7fc, ccb:0x3ffcf87c/0x3ffcf87c
W (20061) BT_AVCT: 2: aloc:0, lcb:0x0/0x3ffcf7fc, ccb:0x3ffcf894/0x3ffcf87c
I (20081) WeiLeng-Player: bt连接状态: 0, 已断开
W (20521) BT_HCI: hci cmd send: sniff: hdl 0x81, intv(400 800)
W (20521) BT_HCI: hcif mode change: hdl 0x81, mode 0, intv 0, status 0x0
W (20531) BT_HCI: hcif mode change: hdl 0x81, mode 2, intv 768, status 0x0
W (24071) BT_HCI: hci cmd send: disconnect: hdl 0x81, rsn:0x13
W (24361) BT_HCI: hcif mode change: hdl 0x81, mode 0, intv 0, status 0x0
W (24441) BT_HCI: hcif disc complete: hdl 0x81, rsn 0x16
```
注意到问题TAG指向：BT_HCI、BT_APPL、BT_AVCT

在menuconfig关掉LHDCV5的选项，再次编译，则蓝牙正常连接，日志是这样：
```c
W (5019) BT_APPL: new conn_srvc id:19, app_id:0
I (5619) WeiLeng-Player: bt连接状态: 2, 已连接
W (8229) BT_HCI: hci cmd send: sniff: hdl 0x81, intv(400 800)
W (8229) BT_HCI: hcif mode change: hdl 0x81, mode 2, intv 768, status 0x0
```
所以后面准备分析一下，在menuconfig启用LHDCV5时，为什么蓝牙连接后会断开；
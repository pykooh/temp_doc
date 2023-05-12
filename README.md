<!-- title: 板内通讯协议 -->

# 说明
## 协议格式, 小端

 * 数据包包结构
 * 小端传输数据
 * [HEAD][LEN][PACKET1][PACKET2]...[CRC16H][CRC16L]
 * head   数据包的头 0x55
 * len    后续的数据包剩余大小
 * crc16  整个数据包的crc16 使用modbus多项式ploy=0x8005 初始值init=0xffff 结果异或值xorout=0
 

## 错误码定义
```c
#define ERROR_CODE_NO_ERROR		                0    //
#define ERROR_CODE_MAINBOARD_PLUG		        1    //主板ntc插口错误
#define ERROR_CODE_MAINBOARD_PCB		        2    //主板ntc错误
#define ERROR_CODE_DISPLAY_NTC1			        3    //显示板ntc错误
#define ERROR_CODE_MAINBOARD_PLUG_OVER          4    //主板ntc插口超温
#define ERROR_CODE_MAINBOARD_PCB_OVER           5    //主板ntc超温
#define ERROR_CODE_DISPLAY_DISCONNECTED			6    //显示断线
#define ERROR_CODE_CONTROL_DISCONNECTED			7    //手控器断线
#define ERROR_CODE_MAINBROAD_DISCONNECTED		8    //主板断线
```

## 包定义
```c {.line-numbers}
//浴霸的基础功能标志位
typedef union _Base_State_T
{
	uint32_t whole;  //只允许清0操作
	struct
	{
		uint8_t on : 1;          //开关
		uint8_t warm1 : 1;       //弱加热
		uint8_t warm2 : 1;       //强加热
		uint8_t bathing : 1;     //洗浴
		uint8_t dry : 1;         //干燥(热干燥
		uint8_t dry_low : 1;     //冷干燥
		uint8_t blow : 1;        //吹风
		uint8_t exhaust : 1;     //换气

		uint8_t swing : 1;       //摆叶
		uint8_t light : 1;       //照明
		uint8_t night_light : 1; //夜灯(氛围灯
		uint8_t clear : 1;       //负离子/净化
        uint8_t timer : 1;       //定时器
		uint8_t timer_config : 1;//定时器设置中
		uint8_t clear_auto : 1;  //自动换气
		uint8_t bluetooth : 1;   //蓝牙音箱
        
		uint8_t audio : 1;       //音效
		uint8_t buzzer_off : 1;  //蜂鸣器关闭(置1的时候,不会触发蜂鸣器)
		uint8_t single_motor : 1;  //单双电机
	}bits;
}Base_State_T;

//浴霸的基础的输出定义
typedef union _Base_Out_T
{
	uint8_t whole;  //只允许清0操作
	struct
	{
		uint8_t warm1 : 1;       //加热1
		uint8_t warm2 : 1;       //加热2
		uint8_t blow : 1;        //吹风
		uint8_t exhaust : 1;     //换气
		uint8_t swing_ready : 1; //摆叶打开
		uint8_t light : 1;       //照明
		uint8_t night_light : 1; //夜灯(氛围灯
		uint8_t clear : 1;       //净化,负离子
	}bits;
}Base_Out_T;


/**
 * 数据包包结构
 * 小端传输数据
 * [HEAD][LEN][DATA...][DATA...][CRC16H][CRC16L]
 * head   数据包的头 0x55
 * len    后续的数据包剩余大小
 * crc16  整个数据包的crc16 使用modbus多项式ploy=0x8005 初始值init=0xffff 结果异或值xorout=0
 */

#define PACKET_HEAD             0x55   //数据包头

#define PACKET_LENGTH_POS       1
#define PACKET_DATA_START_POS   2      //数据开始的位置
#define PACKET_CRC16_POS        2      //数据开始的位置

#define PACKET_CRC_LEN          2      //crc的长度


//------------开关到主板的通信包--------------------------
#define CMD_CONTROL_TO_MAINBOARD_47 0x47
typedef struct __attribute__((packed))
{
	uint8_t       packet_size;       //sizeof(packet_c2m_id47_t)
	uint8_t       cmd_id;            //cmd_id = 0x47
	Base_State_T  state;             //32位
	uint8_t       blow_sec_config;   //设置多少秒的吹风延时
	uint16_t      working_sec;       //当前功能的已经持续时间
	int8_t        control_temper;    //开关的温度 (-25~107)
    uint16_t      remain_time_sec;     //定时剩余时间(秒)
	int8_t        temperature_config;  //洗浴的温控温度(20-45)
	uint8_t       wifi_status;        //wifi的状态
	uint8_t       error_code;        //错误号 0为无错误
}packet_c2m_id47_t;

//------------主板回应给开关的通信包-------------------------
#define CMD_MAINBOARD_TO_CONTROL_48 0x48
typedef struct __attribute__((packed))
{
	uint8_t       packet_size;       //sizeof(packet_m2c_id48_t)
	uint8_t       cmd_id;            //cmd_id = 0x48
	Base_Out_T    real_out;          //8位 真实输出
	uint8_t       blow_sec;          //当前的吹风延时时间
	int8_t        mainboard_temper;  //主板自己的温度 (-25~107)
	int8_t        display_temper;    //显示板的温度   (-25~107)
	uint8_t       error_code;        //错误号 0为无错误
}packet_m2c_id48_t;

//------------主板到显示板的通信包--------------------------
#define CMD_MAINBOARD_TO_DISPLAY_49 0x49
typedef struct __attribute__((packed))
{
	uint8_t       packet_size;         //sizeof(packet_m2d_id49_t)
	uint8_t       cmd_id;              //cmd_id = 0x49
	Base_State_T  flags;               //32位
	uint8_t       blow_sec;            //当前的吹风延时时间
	uint16_t      working_sec;         //当前功能的已经持续时间
	int8_t        control_temper;      //开关的温度 (-25~107)
    uint16_t      remain_time_sec;     //定时剩余时间(秒)
	uint8_t       wifi_status;         //wifi的状态
	uint8_t       error_code;          //错误号 0为无错误
}packet_m2d_id49_t;

//------------显示板到主板的通信包--------------------------
#define CMD_DISPLAY_TO_MAINBOARD_4a 0x4a
typedef struct __attribute__((packed))
{
	uint8_t       packet_size;              //sizeof(packet_d2m_id4a_t)
	uint8_t       cmd_id;                   //cmd_id = 0x4a
	struct 
    {
		uint8_t radar : 1;				    //雷达是否感应到人
		uint8_t swing_ready : 1;            //摆叶已打开
		uint8_t swing_enable : 1;           //显示板没有摆叶
		uint8_t air_quality_level : 2;      //空气质量 00优 01良 02可 03差
    };             
    int8_t        display_temper;      //显示板的温度
	uint8_t       error_code;          //错误号 0为无错误
}packet_d2m_id4a_t;

//------------开关到主板的语音通信包--------------------------
//发一个就响一次
#define CMD_VocieCmd_4b 0x4b
typedef struct __attribute__((packed))
{
	uint8_t       packet_size;
	uint8_t       cmd_id;            //cmd_id = 0x4b
	uint8_t       voice_cmd;         //蓝牙音箱的命令
	uint8_t       voice_val;         
}packet_c2m_voice_id4b_t;
```

## crc16计算
```c
static const uint16_t wCRCTable[] = {
	0x0000, 0xC0C1, 0xC181, 0x0140, 0xC301, 0x03C0, 0x0280, 0xC241,
	0xC601, 0x06C0, 0x0780, 0xC741, 0x0500, 0xC5C1, 0xC481, 0x0440,
	0xCC01, 0x0CC0, 0x0D80, 0xCD41, 0x0F00, 0xCFC1, 0xCE81, 0x0E40,
	0x0A00, 0xCAC1, 0xCB81, 0x0B40, 0xC901, 0x09C0, 0x0880, 0xC841,
	0xD801, 0x18C0, 0x1980, 0xD941, 0x1B00, 0xDBC1, 0xDA81, 0x1A40,
	0x1E00, 0xDEC1, 0xDF81, 0x1F40, 0xDD01, 0x1DC0, 0x1C80, 0xDC41,
	0x1400, 0xD4C1, 0xD581, 0x1540, 0xD701, 0x17C0, 0x1680, 0xD641,
	0xD201, 0x12C0, 0x1380, 0xD341, 0x1100, 0xD1C1, 0xD081, 0x1040,
	0xF001, 0x30C0, 0x3180, 0xF141, 0x3300, 0xF3C1, 0xF281, 0x3240,
	0x3600, 0xF6C1, 0xF781, 0x3740, 0xF501, 0x35C0, 0x3480, 0xF441,
	0x3C00, 0xFCC1, 0xFD81, 0x3D40, 0xFF01, 0x3FC0, 0x3E80, 0xFE41,
	0xFA01, 0x3AC0, 0x3B80, 0xFB41, 0x3900, 0xF9C1, 0xF881, 0x3840,
	0x2800, 0xE8C1, 0xE981, 0x2940, 0xEB01, 0x2BC0, 0x2A80, 0xEA41,
	0xEE01, 0x2EC0, 0x2F80, 0xEF41, 0x2D00, 0xEDC1, 0xEC81, 0x2C40,
	0xE401, 0x24C0, 0x2580, 0xE541, 0x2700, 0xE7C1, 0xE681, 0x2640,
	0x2200, 0xE2C1, 0xE381, 0x2340, 0xE101, 0x21C0, 0x2080, 0xE041,
	0xA001, 0x60C0, 0x6180, 0xA141, 0x6300, 0xA3C1, 0xA281, 0x6240,
	0x6600, 0xA6C1, 0xA781, 0x6740, 0xA501, 0x65C0, 0x6480, 0xA441,
	0x6C00, 0xACC1, 0xAD81, 0x6D40, 0xAF01, 0x6FC0, 0x6E80, 0xAE41,
	0xAA01, 0x6AC0, 0x6B80, 0xAB41, 0x6900, 0xA9C1, 0xA881, 0x6840,
	0x7800, 0xB8C1, 0xB981, 0x7940, 0xBB01, 0x7BC0, 0x7A80, 0xBA41,
	0xBE01, 0x7EC0, 0x7F80, 0xBF41, 0x7D00, 0xBDC1, 0xBC81, 0x7C40,
	0xB401, 0x74C0, 0x7580, 0xB541, 0x7700, 0xB7C1, 0xB681, 0x7640,
	0x7200, 0xB2C1, 0xB381, 0x7340, 0xB101, 0x71C0, 0x7080, 0xB041,
	0x5000, 0x90C1, 0x9181, 0x5140, 0x9301, 0x53C0, 0x5280, 0x9241,
	0x9601, 0x56C0, 0x5780, 0x9741, 0x5500, 0x95C1, 0x9481, 0x5440,
	0x9C01, 0x5CC0, 0x5D80, 0x9D41, 0x5F00, 0x9FC1, 0x9E81, 0x5E40,
	0x5A00, 0x9AC1, 0x9B81, 0x5B40, 0x9901, 0x59C0, 0x5880, 0x9841,
	0x8801, 0x48C0, 0x4980, 0x8941, 0x4B00, 0x8BC1, 0x8A81, 0x4A40,
	0x4E00, 0x8EC1, 0x8F81, 0x4F40, 0x8D01, 0x4DC0, 0x4C80, 0x8C41,
	0x4400, 0x84C1, 0x8581, 0x4540, 0x8701, 0x47C0, 0x4680, 0x8641,
	0x8201, 0x42C0, 0x4380, 0x8341, 0x4100, 0x81C1, 0x8081, 0x4040 };

/**
* @brief 通用crc16计算 POLY为种子
* @param data_p
* @param length
*/
uint16_t crc16(const void *data_p, uint8_t length) 
{
	uint8_t nTemp;
	const uint8_t* nData = (const uint8_t*)data_p;
	uint16_t wCRCuint16_t = 0xFFFF;

	while (length--)
	{
		nTemp = *nData++ ^ wCRCuint16_t;
		wCRCuint16_t >>= 8;
		wCRCuint16_t ^= wCRCTable[(nTemp & 0xFF)];
	}
	return wCRCuint16_t;
}
```


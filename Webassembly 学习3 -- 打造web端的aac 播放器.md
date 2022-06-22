# Webassembly 学习3 -- 打造web端的aac 播放器

## 1、引言 ##

aac 是很常见的音频格式，压缩率比mp3 还高，H5 支持从audio 标签文件读取aac 文件并播放，但不支持从网络流中直接读取。这里借助webassembly 技术，将aac 转码成pcm码流，再借助web audio api 实现aac音频播放。主要用到的开源库有faad、pcm-player

## 2、编译 ##

进入faad 官网，http://www.linuxfromscratch.org/blfs/view/svn/multimedia/faad2.html，下载faad源码包并解压，cd 到工程目录

```
cd faad2-2_10_0/
mkdir dist
emconfigure ./configure --prefix=$(pwd)/dist --disable-shared --without-xmms --without-drm --without-mpeg4ip
emmake make
make install
```

make install 完成后会在dist 目录生成lib、include 等目录，里面存放了faad 头文件和库，复制到路径备用。

## 3、faad 使用 ##

一般来说，我不会直接js 调用faad 库api，而是在外面封装一层，只暴露一些必要的接口。

 **NeAACDecOpen**

 创建一个编码器并返回一个NeAACDecHandle类型句柄

```
NeAACDecHandle NeAACDecOpen(void)

NeAACDecClose(NeAACDecHandle );
```

  **NeAACDecInit2**

  初始化函数，小于0表示失败

```
NEAACDECAPI char NeAACDecInit2(NeAACDecHandle hDecoder,
                               unsigned char *pBuffer,
                               unsigned long SizeOfDecoderSpecificInfo,
                               unsigned long *samplerate,
                               unsigned char *channels);
```

**参数**

hDecoder:	NeAACDecOpen 创建的句柄

pBuffer:		指向缓存空间，用于保存转换后的pcm 数据

SizeOfDecoderSpecificInfo: 输入aac 音频的大小

sampleRate: 		音频采样率

channels:  		音频通道数

**NeAACDecDecode**

```
void* NeAACDecDecode(NeAACDecHandle hpDecoder,
                                 NeAACDecFrameInfo *hInfo,
                                 unsigned char *buffer,
                                 unsigned long buffer_size)
```

hpDecoder:		NeAACDecOpen创建的句柄

NeAACDecFrameInfo:	NeAACDecFrameInfo 指针

buffer:			指向输入AAC音频的缓存区

buffer_size:	输入AAC音频大小

**NeAACDecClose**

	void NeAACDecClose(NeAACDecHandle hpDecoder)

C端完整代码

```
#include <memory.h>
#include <stdlib.h>
#include "faad.h"
#include <stdbool.h>
#include <string.h>
#include <emscripten.h>
#include <stdio.h>
#include <sys/time.h>
#include <sys/timeb.h>
#include <unistd.h>

bool hasInit = false;

NeAACDecHandle decoder = 0;
NeAACDecFrameInfo frame_info;

void PrintArry(unsigned char *buffer, unsigned int size)
{
    int i;
    char data[1024*1024];

    for(i = 0;i < size;i++)
    {
        data[i] = buffer[i];
    }
    
    data[i + 1] = '\0';

}

int init_decoder(unsigned char* inBuffer, size_t size)
{  
    unsigned char channels;
    unsigned long sampleRate;

    memset(&frame_info, 0, sizeof(frame_info));
    decoder = NeAACDecOpen();
    NeAACDecInit(decoder, inBuffer, size, &sampleRate, &channels);
    //printf("init_decoder初始化完毕\n");
    hasInit = true;
    return 0;

}

int feedData(unsigned char* out_data, unsigned char* buffer, unsigned int size)
{
    int ret = 0;

    if (!hasInit)
    {
        init_decoder(buffer, size);
    }
    
    unsigned char *out_buffer = (unsigned char*)NeAACDecDecode(decoder, &frame_info, buffer, size);
    //printf("frame_info.error %d\n",frame_info.error);
    
    if (frame_info.error > 0)
    {        
        return frame_info.error;
    }
    else if(out_buffer && frame_info.samples > 0)//解码成功
    {
        ret = frame_info.samples * frame_info.channels;
        for(int i = 0;i < ret;i++)
        {
             out_data[i] = out_buffer[i];
        }
    }
    
    return ret;

}

void destroyDecoder()
{
    hasInit = false;
    NeAACDecClose(decoder);
}
```

## 4、html 调用 ##

再次进行编译，将C编译成 最终目标wasm

```
export TOTAL_MEMORY=10485760
rm *.js *.wasm

export EXPORTED_FUNCTIONS="[      \
    '_malloc' \
    ,'_free' \
    ,'_destroyDecoder' \
    ,'_feedData'   \
]"

export LIBRARY_FUNCTIONS="[\
    'malloc', \
    'free'      \
]"

# 

emcc aac.c  \
-O3 \
-s WASM=1 \
-I /usr/local -lfaad -lm -L ./lib  \
-s TOTAL_MEMORY=${TOTAL_MEMORY} \
-s DEFAULT_LIBRARY_FUNCS_TO_INCLUDE="${LIBRARY_FUNCTIONS}" \
-s EXPORTED_FUNCTIONS="${EXPORTED_FUNCTIONS}" \
-o aac.js
```

最后html 调用

```
<html>
    <head>
        <link rel="shortcut icon" href="#">
    </head>
    <script type="text/javascript" src="pcm-player.js"></script>
    <script type="text/javascript" src="helper.js"></script>
    <script src="https://cdn.bootcss.com/vConsole/3.2.0/vconsole.min.js"></script>

    <body>
        请点击一下屏幕，才能播放声音
    </body>
    <script>
    
        var decodeCount = 1;
        var isFinish = false;
        var socketURL = "ws://192.168.11.66:9101";
        socketURL = "ws://127.0.0.1:8080";
    
        let vConsole = new VConsole();
    
        var player = new PCMPlayer({
        encoding: '16bitInt',
        channels: 2,
        sampleRate: 44100,
        flushingTime: 22,
        debug:false
        });
    
        FaadModule = {};
        FaadModule.onRuntimeInitialized = function() 
        {
            console.log("Wasm 加载成功!")
            isFinish = true;
        }
    
        function closeDecoder()//关闭解码器
        {
            FaadModule._destroyDecoder();
        }
    
        function decodeAAC(data)
        {
            var retPtr = FaadModule._malloc(4 * 5 * 1024);//接收的数据
            var inputPtr = FaadModule._malloc(4 * data.length);//输入数据
    
            for( i =0;i < data.length;i++)
            {
                FaadModule.HEAPU8[(inputPtr)+i] = data[i];//转换为堆数据
            }
    
            var pcmLen = FaadModule._feedData(retPtr, inputPtr, data.length);
    
            if(pcmLen >= 0)
            {
                //console.log("%d帧 aac 解码成功, %d", decodeCount, pcmLen);
                var pcmData = new Uint8Array(pcmLen);        
                for(i = 0;i < pcmLen;i++)
                {
                    pcmData[i] = FaadModule.HEAPU8[(retPtr)+i]
                }
    
                player.feed(pcmData);
            }
            else
            {
                console.log("%d帧 aac 解码失败, %d", decodeCount, pcmLen);
            }
    
            decodeCount++;
            FaadModule._free(inputPtr);
            FaadModule._free(retPtr);
        }
    
        var ws = new WebSocket(socketURL);
        ws.binaryType = 'arraybuffer';
    
        ws.addEventListener('open', function (event) {
        console.log("发送配置帧");
        ws.send(ConfigChannel("RK3923C1201900139"));
    });
    
        ws.addEventListener('message',function(event) 
        {    
            var input = new Uint8Array(event.data);
            if(input[0] == 0xff)
            {
                //console.log("音频数据");
                if(isFinish)
                {
                    var time = new Date().getTime();
                    decodeAAC(input);
                    //console.log("音频解码耗时:%d ms", new Date().getTime() - time);
                }
    
            }
            else
            {
                //console.log("视频数据");
            }
    
        }
        );
    
    </script>
    <script type="text/javascript" src="aac.js"></script>

</html>
```

**注： chrome 浏览器可能需要点击以下才有声音，跟web-audio api有关 有关。**

**附件：web端播放aac音频**

————————————————

版权声明：本文为CSDN博主「石走刀口」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/sinat_27720649/article/details/115704629

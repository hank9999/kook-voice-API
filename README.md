# 开黑啦 V3 语音API

本项目提供 API 使用说明 及 Python 异步实现 

## 使用须知
**注意本 API 由抓包得来，API 可能会随时变动进而失效**    
**您需要得知使用此API会违反 [开黑啦语音软件许可及服务协议](https://www.kaiheila.cn/protocol.html) `3.2.3` 或 `3.2.5` 条款**  
**同时会违反 [开黑啦开发者隐私政策](https://developer.kaiheila.cn/doc/privacy) `数据信息` 或 `滥用` 中的相关条款**  

## API 说明
gateway: `https://www.kaiheila.cn/api/v3/gateway/voice?channel_id={channel_id}`  
先向 `gateway` 请求 `ws` 链接  
然后取到 `ws` 链接之后，依次发 `1.json` 中的 1,2,3,4  
发了3号之后，把返回的数据存起来, 里面的 `id` 要填到4号的 `transportId` 中  
3号返回的data中 有rtp推流用的 `ip` `port` `rtcpPort`  
每一号都需要生成一个随机数id 用随机int替代  
`ws` 记得发协议的保活 `Ping` `ws`掉了语音自动掉  
实现参考 voice_example.py


## Python voice.py 直接引用说明
```python
from voice import Voice

async def playlist_handler():
    ... # handler 实现
    
    # 加入频道
    voice.channel_id = '频道 ID'
    while True:
        if len(voice.rtp_url) != 0:
            rtp_url = voice.rtp_url
            break
        await asyncio.sleep(0.1)
        
    ... # 推流实现
    
    # 结束当前推流
    voice.is_exit = True
    while True:
        if not voice.is_exit:
            break
        await asyncio.sleep(0.1)
        
async def main():
    await asyncio.wait([
        playlist_handler(), # 播放列表处理
        bot.start(), # khl.py 框架
        voice.handler() # voice 处理
    ])
    
if __name__ == '__main__':
    voice = Voice(token) # 初始化 voice, token 为机器人 token
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

## 推流说明
```
ffmpeg推流
ffmpeg -re -loglevel level+info -nostats -stream_loop -1 -i "xxxxx.mp3" -map 0:a:0 -acodec libopus -ab 128k -filter:a volume=0.8 -ac 2 -ar 48000 -f tee [select=a:f=rtp:ssrc=1357:payload_type=100]rtp://xxxx
```

```
用ffmpeg zmq可以实现切歌不掉
ffmpeg -re -nostats -i "xxx.mp3" -acodec libopus -ab 128k -f mpegts zmq:tcp://127.0.0.1:1234

ffmpeg -re -loglevel level+info -nostats -stream_loop -1 -i zmq:tcp://127.0.0.1:1234 -map 0:a:0 -acodec libopus -ab 128k -filter:a volume=0.8 -ac 2 -ar 48000 -f tee [select=a:f=rtp:ssrc=1357:payload_type=100]rtp://xxxx
```
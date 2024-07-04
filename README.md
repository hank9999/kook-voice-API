# 开黑啦 V3 语音API
## KOOK 官方现已提供官方极简API 本项目归档
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
多推直接引用详细说明 和 单推直接引用说明 见下文

## 多推说明
通过 `API` 请求另一个服务器的语音频道 且 保持上一个语音频道的 `ws` 连接, 可实现同时进多个不同服务器的频道 且 可以推送不同的音频流

## Python voice.py 多推 直接引用详细说明

```python
import asyncio
from voice import Voice

playlist: Dict[str, List[Dict]] = {}
# playlist = {'群组ID': [{'channel': '音频频道ID', 'file': '音频文件路径'}]}
playlist_handle_status = {}


# playlist_handle_status = {'群组ID': True} 某个群组已有处理器

# 新建线程处理对应群组的播放列表
class PlayHandler(threading.Thread):
    def __init__(self, guild: str):
        threading.Thread.__init__(self)
        self.guild = guild
        self.voice = Voice(token)

    def run(self):
        print("开始处理：" + self.guild)
        loop_t = asyncio.new_event_loop()
        loop_t.run_until_complete(self.main())
        print("处理完成：" + self.guild)

    async def main(self):
        task_handler = asyncio.create_task(self.handler())
        task_voice_handler = asyncio.create_task(self.voice.handler())
        await asyncio.wait([
            task_handler,  # 播放列表 处理
            task_voice_handler  # voice 处理
        ], return_when='FIRST_COMPLETED')

    async def handler(self):
        global playlist_handle_status
        while True:
            if len(playlist[self.guild]) != 0:
                play_info = playlist[self.guild].pop(0)  # 取出一个
                self.voice.channel_id = play_info['channel']  # 设置 voice 频道ID
                while True:
                    if len(self.voice.rtp_url) != 0:
                        rtp_url = self.voice.rtp_url  # 获取 rtp 推流链接
                        break
                    await asyncio.sleep(0.1)
                audio_path = play_info['file']  # 获取文件路径
                # 开始推流
                command = f'ffmpeg -re -loglevel level+info -nostats -i "{audio_path}" -map 0:a:0 -acodec libopus -ab 128k -filter:a volume=0.8 -ac 2 -ar 48000 -f tee [select=a:f=rtp:ssrc=1357:payload_type=100]{rtp_url}'
                p = await asyncio.create_subprocess_shell(command, shell=True, stdout=subprocess.DEVNULL,
                                                          stderr=subprocess.DEVNULL)
                while True:
                    # 判断当前歌曲是否推流结束
                    if p.returncode is not None:
                        subprocess.Popen("TASKKILL /F /PID {pid} /T".format(pid=p.pid), stdout=subprocess.DEVNULL,
                                         stderr=subprocess.DEVNULL)
                        # voice 结束当前频道
                        self.voice.is_exit = True
                        while True:
                            if not self.voice.is_exit:
                                break
                            await asyncio.sleep(0.1)
                        break
                    await asyncio.sleep(0.1)
            else:
                # 播放列表为空 退出处理器
                playlist_handle_status[self.guild] = False
                return
            await asyncio.sleep(0.1)


async def playlist_handler():
    while True:
        for guild in playlist.keys():
            # playlist_handle_status 用于判断群组是否已有处理器 没有则新建线程处理
            if guild not in playlist_handle_status and len(playlist[guild]) != 0:
                playlist_handle_status[guild] = True
                PlayHandler(guild).start()
            if guild in playlist_handle_status and not playlist_handle_status[guild] and len(playlist[guild]) != 0:
                playlist_handle_status[guild] = True
                PlayHandler(guild).start()
        await asyncio.sleep(0.1)


async def main():
    task_playlist_handler = asyncio.create_task(playlist_handler())
    task_bot = asyncio.create_task(bot.start())
    await asyncio.wait([
        task_playlist_handler,  # 播放列表处理
        task_bot  # khl.py 框架
    ])


if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

## Python voice.py 单推 直接引用说明

```python
import asyncio
from voice import Voice


async def playlist_handler():
    ...  # handler 实现

    # 加入频道
    voice.channel_id = '频道 ID'
    while True:
        if len(voice.rtp_url) != 0:
            rtp_url = voice.rtp_url
            break
        await asyncio.sleep(0.1)

    ...  # 推流实现

    # 结束当前推流
    voice.is_exit = True
    while True:
        if not voice.is_exit:
            break
        await asyncio.sleep(0.1)


async def main():
    task_playlist_handler = asyncio.create_task(playlist_handler())
    task_bot = asyncio.create_task(bot.start())
    task_voice_handler = asyncio.create_task(voice.handler())
    await asyncio.wait([
        task_playlist_handler,  # 播放列表处理
        task_bot,  # khl.py 框架
        task_voice_handler  # voice 处理
    ])


if __name__ == '__main__':
    voice = Voice(token)  # 初始化 voice, token 为机器人 token
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

## 推流说明
```
ffmpeg推流
ffmpeg -re -loglevel level+info -nostats -i "xxxxx.mp3" -map 0:a:0 -acodec libopus -ab 128k -filter:a volume=0.8 -ac 2 -ar 48000 -f tee [select=a:f=rtp:ssrc=1357:payload_type=100]rtp://xxxx
```

```
用ffmpeg zmq可以实现切歌不掉
ffmpeg -re -nostats -i "xxx.mp3" -acodec libopus -ab 128k -f mpegts zmq:tcp://127.0.0.1:1234

ffmpeg -re -loglevel level+info -nostats -stream_loop -1 -i zmq:tcp://127.0.0.1:1234 -map 0:a:0 -acodec libopus -ab 128k -filter:a volume=0.8 -ac 2 -ar 48000 -f tee [select=a:f=rtp:ssrc=1357:payload_type=100]rtp://xxxx
```

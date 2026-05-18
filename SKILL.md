---
name: nonebot-plugin-dev
description: NoneBot2 插件开发完整指南。用于编写、调试、发布 NoneBot2 Python 聊天机器人插件，包括事件响应器、消息处理、依赖注入、定时任务、OneBot V11 适配器等。当用户要求写 NoneBot/nb 插件时使用此技能。
---

# NoneBot2 插件开发技能

> NoneBot2 v2.5.0 | Python >= 3.9 | 异步优先框架

## 目录

1. [核心架构](#核心架构)
2. [插件结构与加载](#插件结构与加载)
3. [事件响应器](#事件响应器)
4. [事件处理函数](#事件处理函数)
5. [依赖注入](#依赖注入)
6. [消息处理](#消息处理)
7. [会话控制](#会话控制)
8. [定时任务](#定时任务)
9. [配置管理](#配置管理)
10. [OneBot V11 适配器](#onebot-v11-适配器)
11. [完整示例](#完整示例)
12. [最佳实践](#最佳实践)

---

## 核心架构

```
Driver (底层通信) → Adapter (协议适配) → Bot (机器人实例)
                                          ↓
Event (事件) → Matcher (事件响应器) → Handler (处理函数)
```

- **Driver**: 底层通信框架（FastAPI/Quart/httpx 等）
- **Adapter**: 协议适配器（OneBot V11/V12、Telegram、Discord 等）
- **Plugin**: 插件 = Python 模块，包含 Matcher + Handler
- **Matcher**: 事件响应器，按规则筛选事件
- **依赖注入**: 通过类型注解自动注入 Bot/Event/State 等上下文

---

## 插件结构与加载

### 单文件插件

```
plugins/
└── my_plugin.py
```

### 包插件（推荐）

```
plugins/
└── my_plugin/
    ├── __init__.py      # 插件入口
    └── config.py        # 配置类（可选）
```

### 加载方式

```python
import nonebot

nonebot.init()

# 方式1：加载目录下所有插件
nonebot.load_plugins("awesome_bot/plugins")

# 方式2：按模块名加载
nonebot.load_plugin("nonebot_plugin_xxx")

# 方式3：加载内置插件
nonebot.load_builtin_plugins("echo")

nonebot.run()
```

### pyproject.toml 配置加载

```toml
[tool.nonebot]
plugin_dirs = ["awesome_bot/plugins"]

[tool.nonebot.plugins]
"nonebot-plugin-apscheduler" = ["nonebot_plugin_apscheduler"]
```

---

## 事件响应器

### 辅助函数一览

```python
from nonebot import (
    on,              # 任意类型
    on_metaevent,    # 元事件
    on_message,      # 消息事件
    on_notice,       # 通知事件
    on_request,      # 请求事件
    on_command,      # 命令匹配（最常用）
    on_shell_command,# shell 风格命令
    on_startswith,   # 前缀匹配
    on_endswith,     # 后缀匹配
    on_fullmatch,    # 完全匹配
    on_keyword,      # 关键词匹配
    on_regex,        # 正则匹配
)
```

### 常用参数

```python
matcher = on_command(
    "天气",                    # 命令名
    rule=to_me(),             # 响应规则（需要@bot）
    aliases={"weather", "查天气"},  # 命令别名
    priority=10,              # 优先级（越小越优先）
    block=True,               # 是否阻断后续响应器
    permission=SUPERUSER,     # 权限控制
)
```

### 内置响应规则

```python
from nonebot.rule import (
    to_me,        # @bot 或私聊
    startswith,   # 消息开头匹配
    endswith,     # 消息结尾匹配
    fullmatch,    # 完全匹配
    keyword,      # 包含关键词
    command,      # 命令匹配
    regex,        # 正则匹配
    is_type,      # 事件类型匹配
)
```

### 命令组

```python
from nonebot import CommandGroup

group = CommandGroup("管理", priority=5)
ban_cmd = group.command("封禁")      # 匹配 /管理.封禁
kick_cmd = group.command("踢出")     # 匹配 /管理.踢出
```

### 响应器组

```python
from nonebot import MatcherGroup

group = MatcherGroup(rule=to_me(), priority=5)
m1 = group.on_command("a")
m2 = group.on_command("b")
```

---

## 事件处理函数

### 基本用法

```python
from nonebot import on_command

weather = on_command("天气", priority=10, block=True)

@weather.handle()
async def handle_first():
    """第一个处理函数"""
    await weather.finish("今天天气晴朗")
```

### 多步处理流程

```python
@weather.handle()
async def handle_first(args: Message = CommandArg()):
    if args.extract_plain_text():
        # 有参数，直接处理
        pass
    else:
        await weather.pause("请输入城市名")  # 暂停等待下一条消息

@weather.handle()
async def handle_city(event: Event):
    city = event.get_plaintext()
    await weather.finish(f"{city}的天气是...")
```

### 事件响应器操作

```python
await matcher.send("消息")      # 发送消息，不结束
await matcher.finish("消息")    # 发送消息并结束流程
await matcher.pause("消息")     # 发送消息并等待下一条
await matcher.reject("消息")    # 拒绝并重新等待（用于 got）
await matcher.skip()            # 跳过当前 handler
matcher.stop_propagation()      # 阻止事件传播到下一优先级
```

> ⚠️ `finish`/`pause`/`reject`/`skip` 通过抛异常实现，后续代码不会执行。
> 在 try-except 中务必排除 `MatcherException`：

```python
from nonebot.exception import MatcherException

try:
    await matcher.finish("done")
except MatcherException:
    raise
except Exception as e:
    pass
```

---

## 依赖注入

### 类型注入（直接标注类型）

```python
from nonebot.adapters import Bot, Event, Message
from nonebot.typing import T_State
from nonebot.matcher import Matcher

@matcher.handle()
async def _(
    bot: Bot,           # 当前 Bot 实例
    event: Event,       # 当前事件
    state: T_State,     # 会话状态字典
    matcher: Matcher,   # 当前响应器实例
):
    pass
```

### 参数注入（nonebot.params）

```python
from nonebot.params import (
    CommandArg,      # 命令参数 Message
    Command,         # 命令元组 tuple[str, ...]
    RawCommand,      # 原始命令字符串
    CommandStart,    # 命令前缀
    RegexGroup,      # 正则分组 tuple
    RegexDict,       # 正则命名分组 dict
    RegexStr,        # 正则匹配的完整字符串
    EventType,       # 事件类型 str
    EventMessage,    # 事件消息 Message
    EventPlainText,  # 事件纯文本 str
    EventToMe,       # 是否@bot bool
    Depends,         # 子依赖
    ArgPlainText,    # got 获取的参数纯文本
    Arg,             # got 获取的参数 Message
)
```

### 子依赖（Depends）

```python
from nonebot.params import Depends

async def get_user_name(event: Event) -> str:
    return event.get_user_id()

@matcher.handle()
async def _(name: str = Depends(get_user_name)):
    await matcher.finish(f"你好 {name}")
```

### 使用 Annotated（推荐，Python 3.9+）

```python
from typing import Annotated
from nonebot.params import Depends

@matcher.handle()
async def _(name: Annotated[str, Depends(get_user_name)]):
    pass
```

---

## 消息处理

### 消息序列 Message

```python
from nonebot.adapters.onebot.v11 import Message, MessageSegment

# 构造消息
msg = Message("纯文本")
msg = Message([MessageSegment.text("hello"), MessageSegment.image(file="xxx.jpg")])

# 拼接
msg = MessageSegment.text("hello") + MessageSegment.face(1)
msg += " world"

# 提取纯文本
text = msg.extract_plain_text()

# 过滤类型
images = msg["image"]           # 获取所有图片段
first_img = msg["image", 0]    # 第一个图片段

# 遍历
for seg in msg:
    if seg.type == "image":
        url = seg.data.get("url")
```

### OneBot V11 消息段

```python
from nonebot.adapters.onebot.v11 import MessageSegment

MessageSegment.text("文本")
MessageSegment.image(file="https://xxx.jpg")    # 网络图片
MessageSegment.image(file="file:///path.jpg")   # 本地图片
MessageSegment.image(file="base64://...")        # base64 图片
MessageSegment.at(user_id=123456)               # @某人
MessageSegment.at(user_id="all")                # @全体
MessageSegment.face(id=1)                       # QQ 表情
MessageSegment.record(file="xxx.mp3")           # 语音
MessageSegment.video(file="xxx.mp4")            # 视频
MessageSegment.reply(id=msg_id)                 # 回复
MessageSegment.json(data=json_str)              # JSON 卡片
MessageSegment.node_custom(...)                 # 合并转发节点
```

### 消息模板

```python
from nonebot.adapters.onebot.v11 import Message

# 使用模板构造消息
msg = Message.template("{} 你好！今天是 {}").format(
    MessageSegment.at(user_id=123),
    "周一"
)
```

---

## 会话控制

### got 装饰器（多轮对话）

```python
from nonebot import on_command
from nonebot.params import ArgPlainText

weather = on_command("天气")

@weather.handle()
async def _(args: Message = CommandArg()):
    if args.extract_plain_text():
        weather.set_arg("city", args)  # 直接设置参数

@weather.got("city", prompt="请输入城市名")
async def _(city: str = ArgPlainText()):
    if city not in VALID_CITIES:
        await weather.reject("城市名无效，请重新输入")
    await weather.finish(f"{city}天气：晴")
```

### receive 装饰器（等待下一条消息）

```python
@matcher.receive("key")
async def _(event: Event):
    # event 是用户发送的下一条消息事件
    pass
```

---

## 定时任务

需要安装 `nonebot-plugin-apscheduler`。

```python
from nonebot import require, get_bot

require("nonebot_plugin_apscheduler")
from nonebot_plugin_apscheduler import scheduler

# 装饰器方式
@scheduler.scheduled_job("cron", hour=8, minute=0, id="morning_greeting")
async def morning():
    bot = get_bot()
    await bot.send_group_msg(group_id=123456, message="早上好！")

# add_job 方式
scheduler.add_job(
    some_func, "interval", hours=2, id="check_task"
)
```

> ⚠️ 定时任务中**不能**使用依赖注入和 Matcher 操作，需要手动获取 Bot 并调用 API。

---

## 配置管理

### .env 文件

```ini
# .env.prod
DRIVER=~fastapi
HOST=0.0.0.0
PORT=8080
COMMAND_START=["/", "!"]
COMMAND_SEP=["."]
SUPERUSERS=["123456"]

# 自定义配置
MY_API_KEY=xxx
```

### 插件配置类

```python
# config.py
from pydantic import BaseModel

class Config(BaseModel):
    my_api_key: str = ""
    my_timeout: int = 30
```

```python
# __init__.py
from nonebot import get_plugin_config
from .config import Config

config = get_plugin_config(Config)
print(config.my_api_key)
```

---

## OneBot V11 适配器

### 常用事件类型

```python
from nonebot.adapters.onebot.v11 import (
    # 消息事件
    MessageEvent,
    PrivateMessageEvent,
    GroupMessageEvent,
    # 通知事件
    NoticeEvent,
    GroupIncreaseNoticeEvent,
    GroupDecreaseNoticeEvent,
    FriendAddNoticeEvent,
    GroupRecallNoticeEvent,
    PokeNotifyEvent,
    # 请求事件
    RequestEvent,
    FriendRequestEvent,
    GroupRequestEvent,
)
```

### 事件属性（GroupMessageEvent）

```python
event.user_id       # 发送者 QQ
event.group_id      # 群号
event.message_id    # 消息 ID
event.message       # 消息内容 Message
event.raw_message   # 原始消息字符串
event.sender        # 发送者信息（nickname/card/role 等）
event.get_plaintext()  # 纯文本
event.get_user_id()    # 用户 ID 字符串
event.is_tome()        # 是否@bot
```

### 调用 API

```python
# 发送消息
await bot.send(event, "消息内容")
await bot.send_group_msg(group_id=123, message="hello")
await bot.send_private_msg(user_id=456, message="hello")

# 撤回消息
await bot.delete_msg(message_id=msg_id)

# 群管理
await bot.set_group_ban(group_id=123, user_id=456, duration=60)
await bot.set_group_kick(group_id=123, user_id=456)
await bot.set_group_admin(group_id=123, user_id=456, enable=True)
await bot.set_group_card(group_id=123, user_id=456, card="新名片")

# 获取信息
info = await bot.get_group_member_info(group_id=123, user_id=456)
group_list = await bot.get_group_list()
member_list = await bot.get_group_member_list(group_id=123)

# 合并转发
await bot.send_group_forward_msg(group_id=123, messages=[
    {"type": "node", "data": {"name": "Bot", "uin": "10001", "content": "消息1"}},
    {"type": "node", "data": {"name": "Bot", "uin": "10001", "content": "消息2"}},
])
```

### 权限控制

```python
from nonebot.permission import SUPERUSER
from nonebot.adapters.onebot.v11 import (
    GROUP_ADMIN,    # 群管理员
    GROUP_OWNER,    # 群主
    GROUP_MEMBER,   # 群成员
    PRIVATE_FRIEND, # 好友私聊
)

admin_cmd = on_command("ban", permission=SUPERUSER | GROUP_ADMIN | GROUP_OWNER)
```

---

## 完整示例

### 示例1：天气查询插件

```python
"""天气查询插件"""
from nonebot import on_command
from nonebot.rule import to_me
from nonebot.adapters import Message
from nonebot.params import CommandArg, ArgPlainText
from nonebot.adapters.onebot.v11 import GroupMessageEvent

weather = on_command("天气", rule=to_me(), priority=10, block=True, aliases={"查天气"})

@weather.handle()
async def handle_first(args: Message = CommandArg()):
    if args.extract_plain_text():
        weather.set_arg("city", args)

@weather.got("city", prompt="请输入要查询的城市名")
async def handle_city(event: GroupMessageEvent, city: str = ArgPlainText()):
    if not city:
        await weather.reject("城市名不能为空，请重新输入")
    # 这里调用天气 API
    result = f"🌤 {city}今日天气：晴，25°C"
    await weather.finish(result)
```

### 示例2：关键词自动回复

```python
"""关键词回复插件"""
from nonebot import on_keyword
from nonebot.adapters.onebot.v11 import GroupMessageEvent, MessageSegment

hello = on_keyword({"你好", "hello", "hi"}, priority=50, block=False)

@hello.handle()
async def _(event: GroupMessageEvent):
    await hello.send(
        MessageSegment.at(event.user_id) + MessageSegment.text(" 你好呀~")
    )
```

### 示例3：正则匹配 + 图片发送

```python
"""表情包插件"""
import re
from nonebot import on_regex
from nonebot.params import RegexGroup
from nonebot.adapters.onebot.v11 import MessageSegment

emoji = on_regex(r"^发表情\s*(.+)$", priority=20, block=True)

@emoji.handle()
async def _(matched: tuple = RegexGroup()):
    name = matched[0]
    url = f"https://api.example.com/emoji/{name}.gif"
    await emoji.finish(MessageSegment.image(file=url))
```

### 示例4：定时推送

```python
"""每日推送插件"""
from nonebot import require, get_bot

require("nonebot_plugin_apscheduler")
from nonebot_plugin_apscheduler import scheduler

PUSH_GROUPS = [123456, 789012]

@scheduler.scheduled_job("cron", hour=8, minute=0, id="daily_push")
async def daily_push():
    bot = get_bot()
    for group_id in PUSH_GROUPS:
        await bot.send_group_msg(group_id=group_id, message="☀️ 早上好！新的一天开始了~")
```

### 示例5：带配置的完整包插件

```
plugins/my_plugin/
├── __init__.py
└── config.py
```

```python
# config.py
from pydantic import BaseModel

class Config(BaseModel):
    my_plugin_api_url: str = "https://api.example.com"
    my_plugin_timeout: int = 30
```

```python
# __init__.py
from nonebot import on_command, get_plugin_config
from nonebot.plugin import PluginMetadata
from nonebot.adapters import Message
from nonebot.params import CommandArg
import httpx

from .config import Config

__plugin_meta__ = PluginMetadata(
    name="我的插件",
    description="一个示例插件",
    usage="/查询 <关键词>",
    config=Config,
)

config = get_plugin_config(Config)

query = on_command("查询", priority=10, block=True)

@query.handle()
async def _(args: Message = CommandArg()):
    keyword = args.extract_plain_text().strip()
    if not keyword:
        await query.finish("请输入查询关键词")

    async with httpx.AsyncClient(timeout=config.my_plugin_timeout) as client:
        resp = await client.get(f"{config.my_plugin_api_url}/search", params={"q": keyword})
        data = resp.json()

    await query.finish(f"查询结果：{data.get('result', '无')}")
```

---

## 最佳实践

### 插件元数据

```python
from nonebot.plugin import PluginMetadata

__plugin_meta__ = PluginMetadata(
    name="插件名称",
    description="插件描述",
    usage="使用说明",
    type="application",  # application/library
    config=Config,       # 配置类
    homepage="https://github.com/xxx",
)
```

### 跨插件依赖

```python
from nonebot import require

# 确保依赖插件已加载
require("nonebot_plugin_xxx")
from nonebot_plugin_xxx import some_function
```

### 错误处理

```python
from nonebot.exception import MatcherException, ActionFailed

@matcher.handle()
async def _(bot: Bot, event: Event):
    try:
        await bot.send_group_msg(group_id=123, message="test")
    except ActionFailed as e:
        # API 调用失败（被风控、权限不足等）
        logger.error(f"发送失败: {e}")
    except MatcherException:
        raise  # 不要捕获 Matcher 异常
    except Exception as e:
        logger.error(f"未知错误: {e}")
```

### 日志

```python
from nonebot import logger

logger.info("信息")
logger.warning("警告")
logger.error("错误")
logger.debug("调试")
```

### 重载（按事件子类型分发）

```python
from nonebot.adapters.onebot.v11 import PrivateMessageEvent, GroupMessageEvent

@matcher.handle()
async def handle_private(event: PrivateMessageEvent):
    """只处理私聊"""
    await matcher.finish("这是私聊回复")

@matcher.handle()
async def handle_group(event: GroupMessageEvent):
    """只处理群聊"""
    await matcher.finish("这是群聊回复")
```

### 网络请求

```python
import httpx

async with httpx.AsyncClient() as client:
    resp = await client.get("https://api.example.com/data")
    data = resp.json()
```

### 数据持久化

```python
import json
from pathlib import Path

DATA_FILE = Path(__file__).parent / "data.json"

def load_data() -> dict:
    if DATA_FILE.exists():
        return json.loads(DATA_FILE.read_text())
    return {}

def save_data(data: dict):
    DATA_FILE.write_text(json.dumps(data, ensure_ascii=False, indent=2))
```

---

## 关键注意事项

1. **异步优先**：所有 handler 推荐用 `async def`，网络请求用 `httpx`/`aiohttp`
2. **finish 会抛异常**：`finish`/`pause`/`reject` 后的代码不会执行
3. **block=True**：命令类响应器建议设置，防止被低优先级响应器重复处理
4. **priority 越小越优先**：重要命令用低数字
5. **不要在插件加载前 import 插件模块**
6. **定时任务不能用依赖注入**：需手动 `get_bot()` + 调 API
7. **OneBot V11 消息段类型**：text/image/face/at/reply/record/video/json/forward
8. **配置前缀**：自定义配置建议加插件名前缀避免冲突
9. **require 跨插件**：使用其他插件功能前必须 `require()`
10. **消息段是适配器特定的**：必须从对应适配器导入 `Message`/`MessageSegment`

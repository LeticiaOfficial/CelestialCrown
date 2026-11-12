# CelestialCrown 星冠SDK (Docs Only)

QQ NT 的本地桥接器，用 Python 调用客户端的消息、好友和群组接口。

项目由 C 原生模块、JavaScript 桥服务和 Python SDK 组成，面向 Windows x64，适配版本为 **QQ NT 9.9.37-54395**。消息收发、联系人读取和群管理都有对应的功能封装；接口索引收录了 **78 个服务、2759 个方法**。

本仓库用于展示项目，只提供这份说明。代码片段用于介绍接口和调用流程，源码与运行包暂未公开。

> 如果你只是想“养鱼”，请移步 [QQ 官方机器人平台](https://q.qq.com/)，或查看 [QQ 机器人官方接入文档](https://bot.q.qq.com/wiki/)，选择适合自己的官方接入方式与配套工具。
>
> **本项目不保证账号安全。** 即使仅在本机、低频使用，也可能触发风控，造成账号限制、冻结或封禁。项目不提供防封、解封或账号恢复承诺。

[使用手册](https://graniterstudio.com/sharing) · [功能](#features) · [技术栈](#stack) · [调用示例](#examples) · [版本与限制](#limits) · [下载与获取](#download) · [使用许可](#license) · [写在最后的一封信](#letter)

<a id="features"></a>
## 功能

| 分类 | 内容 |
| --- | --- |
| 消息 | 私聊与群聊文本、图片、文件发送，消息转发与撤回 |
| 消息读取 | 近期消息聚合 |
| 入站监听 | 外部私聊、群聊文本消息回调，通过 poll() 手动轮询 |
| 好友与资料 | 好友列表、已知昵称、资料查询、好友申请与删除 |
| 群组 | 群列表、群成员 UID |
| 群管理 | 单人禁言、全员禁言、移出成员、邀请成员、修改群名 |
| 表情 | 收藏表情读取、添加、删除和发送 |
| 服务列表 | 读取本地索引中的服务名称 |

以上仅列出已有直接验证依据的功能范围。实际结果取决于客户端版本、登录状态、账号权限和具体参数。2759 是索引收录的方法数，不代表每个方法都已验证可用。

<a id="stack"></a>
## 技术栈

| 部分 | 技术 | 职责 |
| --- | --- | --- |
| 原生模块 | C、Win32 API、MSVC x64 | 在客户端进程内加载桥脚本，与运行时衔接 |
| 桥服务 | JavaScript、客户端内置的 Node.js / Electron | 接收本机请求，调用客户端接口，收集事件与推送 |
| 本地通信 | HTTP、JSON、Token 鉴权 | 连接 Python 客户端和桥服务 |
| Python SDK | Python 3.10+ 标准库 | 提供按功能组织的 API、参数检查、结果解析 |
| 接口索引 | JSON | 保存服务、方法、参数数量和已知状态，供检索与动态 SDK 使用 |
| 构建与运维 | MSVC、批处理、PowerShell | 原生模块构建、部署、重启与还原 |

Python 客户端不需要安装第三方包。桥服务使用 QQ 客户端自带的 JavaScript 运行时，不要求用户另外安装 Node.js。

调用关系如下：

```text
Python 脚本
    ↓ SDK / HTTP / JSON
本地桥服务
    ↓ 客户端接口
QQ NT
```

常用功能通过 `qq.api` 调用，例如 `qq.api.msg.send()` 和 `qq.api.group.list()`。`qq.sdk` 则按服务与方法提供动态访问入口；其中的方法仍需要正确的参数和适用的客户端环境。

<a id="examples"></a>
## 调用示例

以下示例按现有 Python SDK 的方法签名编写，仅作接口展示。它们依赖另行提供的客户端代码、已经部署的桥服务及已登录的 QQ，不能在这个只含 README 的仓库中直接运行。

示例中的 `u_EXAMPLE_FRIEND`、`u_EXAMPLE_MEMBER`、QQ 号、群号、消息 ID 和文件路径均为占位值。

### 1. 从哪开始

先申请并取得对应环境的交付材料，确认客户端与本机桥已就绪，再使用星冠的 Python 入口。此仓库没有可运行包，也没有公开的 PyPI 安装命令。

SDK 的基本入口：

```python
from celestialcrown import CelestialCrown

qq = CelestialCrown(retries=0)
print(qq.api.system.ping())
```

客户端默认连接 `http://127.0.0.1:7847`，并尝试从本机环境或令牌文件读取 Token。也可以显式指定地址和 Token：

```python
qq = CelestialCrown(
    base="http://127.0.0.1:7847",
    token="此处填写本机桥服务的 Token",
    timeout=30.0,
    retries=0,
)
```

`ping()` 用于检查桥服务是否存活；能连上桥不代表 QQ 已经登录。联系人、群组和消息接口还需要客户端登录态。

调用前需要分清三种标识：

| 标识 | 示例 | 用途 |
| --- | --- | --- |
| UID | `u_EXAMPLE_FRIEND` | 好友或成员的内部标识，私聊发送使用它 |
| UIN | `1000000001` | QQ 号，部分资料查询使用它 |
| 群号 | `123456789` | 群聊目标；调用消息接口时配合 `chat_type=2` |

读取已知好友 UID 与昵称：

```python
friends = qq.api.buddy.list(with_names=True)

for uid in friends.get("uids", []):
    print(uid, friends.get("names", {}).get(uid, "暂时没有名字"))
```

昵称来自已知资料，可能不完整。`retries=0` 用于避免额外的 HTTP 重试；兼容环境变量 `NTQQ_BRIDGE_RETRIES` 若已设置，也需要为 `0`，否则会覆盖该参数。

### 2. 发送文本、图片与文件

私聊使用内部 UID：

```python
friend_uid = "u_EXAMPLE_FRIEND"

result = qq.api.msg.send(
    friend_uid,
    "你好，这是一条测试消息。",
    allow_send=True,
)
print(result)
```

群聊使用群号，并指定会话类型：

```python
group_code = "123456789"

result = qq.api.msg.send(
    group_code,
    "群消息示例。",
    chat_type=2,
    allow_send=True,
)
print(result)
```

图片和文件使用各自的发送方法：

```python
image_result = qq.api.msg.send_image(
    friend_uid,
    r"D:\Images\example.png",
    allow_send=True,
)

file_result = qq.api.msg.send_file(
    group_code,
    r"D:\Documents\example.pdf",
    chat_type=2,
    allow_send=True,
)

print(image_result)
print(file_result)
```

这两个方法会先将媒体文件复制到 QQ 使用的本地媒体目录，再发起发送。原始文件必须存在，相关目录也需要可写；图片尺寸解析目前处理 PNG 和 JPEG。

发送方法默认要求显式传入 `allow_send=True`。返回中的 `status` 表示调用响应状态，业务结果还要查看 `result`；收到 `full` 不能直接当作消息已送达的证明。

### 3. 查询群组与管理成员

```python
groups = qq.api.group.list()

for group in groups:
    print(group.get("groupCode"), group.get("groupName"))

members = qq.api.group.members("123456789")
print("已读取人数：", members.get("count"))
print("成员 UID：", members.get("uids"))
print("读取完成标记：", members.get("finish"))
```

成员查询主要返回 UID。若需 QQ 号或昵称，应结合已知资料查询，不能把 UID 列表当作完整成员资料。

群管理方法单独放在 `group_admin` 中，例如禁言十分钟：

```python
qq.api.group_admin.mute(
    "123456789",
    ["u_EXAMPLE_MEMBER"],
    seconds=600,
    allow_admin=True,
)

```

`allow_admin=True` 是 SDK 的操作确认参数，不会赋予账号管理员权限。是否允许操作仍由 QQ 的账号权限和服务端规则决定。

### 4. 转发、撤回与收藏表情

将一条私聊消息转发到群聊，需要提供源会话、消息 ID 和目标会话：

```python
qq.api.msg.forward(
    "u_EXAMPLE_FRIEND",
    ["EXAMPLE_MSG_ID"],
    "123456789",
    src_chat_type=1,
    dst_chat_type=2,
    allow_forward=True,
)
```

撤回自己发送的消息：

```python
qq.api.msg.recall(
    "123456789",
    "EXAMPLE_MSG_ID",
    chat_type=2,
    allow_recall=True,
)
```

读取收藏表情，再使用其中的资源 ID 发送：

```python
favorites = qq.api.emoji.favorites(count=48)
print(favorites)

qq.api.emoji.send_favorite(
    "EXAMPLE_RES_ID",
    "u_EXAMPLE_FRIEND",
    allow_send=True,
)
```

消息 ID 和表情资源 ID 都需要来自当前账号的实际数据。转发与撤回还受消息状态、权限和客户端规则限制。

### 5. 接收私聊与群消息

私聊与群聊入站路径已有实机记录：外部账号发送文本时，`on_private`、`on_group` 能够触发。群聊的 `peer_uid` / `peer_uin` 都是群号，发言人的 QQ 号看 `sender_uin`。

以下示例只打印内容，不自动回复。保存到交付包根目录后再使用：

```python
import sys
import time
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent / "client"))
from celestialcrown import CelestialCrown

qq = CelestialCrown(retries=0)

@qq.api.listener.on_private
def private_message(message):
    print("私聊", message.sender_uin, message.text)

@qq.api.listener.on_group
def group_message(message):
    print("群聊", message.peer_uid, message.sender_name,
          message.sender_uin, message.text)

end_at = time.monotonic() + 30
while time.monotonic() < end_at:
    qq.api.listener.poll()
    time.sleep(0.3)
```

昵称缺失时会尝试用已知资料或 QQ 号查询补全，但不能保证每条消息都有完整资料。首次轮询可能读到桥缓冲中的旧消息；进程内缓存不提供持久化、断线补发或固定延迟保证。这两条入站记录不用于认证其他监听入口或自动回复。

### 6. 参数与返回结构
- 暂不提供
## 版本与限制

- 当前适配目标为 **Windows x64 / QQ NT 9.9.37-54395**。其他版本未提供兼容保证。
- **不保证账号安全，也不保证不会被封禁。** 本地运行、减少调用或使用自己的账号，都不能消除平台风控风险。
- 客户端需要保持运行并登录。桥服务的存活检查不能代替登录状态检查。
- QQ 空间相关功能尚未打通；商城表情包发送所需参数也未补齐。
- 接口索引包含未测试和已知异常的方法，不提供全量可用承诺。
- 联系人资料和部分列表依赖已收到的推送，可能为空或不完整。
- 此页展示功能与接口形式，不提供安装、部署或运行支持。

<a id="download"></a>
## 下载与获取

**暂未开放下载。**

GitHub 仓库仅保留 README，用于项目展示。后续如提供下载，将通过仓库外的渠道发布，并在这里更新实际入口。

<a id="license"></a>
## 使用许可

本项目采用自定义限制性许可，保留所有未明确授予的权利。

- 本页仅供个人阅读、学习和了解项目。公开展示不代表授予源码、程序或运行包的使用与分发权。
- **禁止商用**：未经作者事先书面许可，不得将本项目及其衍生内容用于销售、付费服务、商业产品、收费代部署或其他营利活动。
- **禁止广泛传播**：未经作者事先书面许可，不得转载整套资料、建立镜像、二次上传、打包转售，或通过网盘、群文件及其他渠道批量分发源码、程序、运行包或衍生版本。
- 可以分享本仓库的原始页面链接。分享链接不包含复制、镜像或重新分发项目内容的授权。
- 后续即使获得下载地址，也不因此取得商业使用、再许可或再次分发的权利；其他用途需另行获得作者书面许可。

项目与腾讯无隶属或授权关系。QQ 及相关名称、商标与客户端程序的权利归各自权利人所有。

---

<a id="letter"></a>
## 写在最后的一封信

写给读到这里的你：

我讨厌这样的循环。一个项目还默默无闻时，愿意了解它的人寥寥无几；等它火起来，流量、索取和滥用便一同涌来。曾经带来无数欢乐的东西，渐渐被推到风口，原本出于兴趣写代码的人，却要替所有人的期待和越界收拾残局。我不愿再看见这样的热闹，以作者的疲惫和项目的沉寂收场。

会做，便被期待做得更多；愿意分享，便被要求一直负责。有人贡献了一段业余时间，后来却仿佛欠下了一份没有期限的工。

我讨厌泛滥的“鱼”，也讨厌那种理所当然的傲慢。有些热闹，看得久了，便只剩疲惫；有些事情，走到最后，谁都说与自己无关。“雪崩的时候，没有一片雪花是无辜的。”可等一切安静下来，曾经值得珍惜的东西，也未必还在原处。

有些项目曾给过许多人折腾、创作和分享的乐趣。后来发生的种种，让我越来越在意：那些曾经习以为常的欢乐，究竟还能留下多少？我记得它们带来的快乐，也记得那些让它们存在的人。

但我更讨厌另一种结局：因为有人滥用，就连同所有人的可能性一起扼杀；因为有人越界，就认定所有好奇心都有罪。我讨厌以秩序之名强行塑造社区，讨厌把每个人都压进同一套模具，也讨厌那些贪得无厌、恨不得吞尽每一寸空间的饕餮。

我希望一个社区容得下不同的兴趣、不同的做法，也容得下一个人只为自己做一点东西。愿意分享的人可以分享，想安静折腾的人也可以安静折腾；每个人都应当知道，自己的自由到哪里会开始伤害别人。

因此，我做了你在这里看到的东西。我想保留独属于自己的那一份欢乐，也给可能身处相同境遇、抱有相同想法的人留下一点思路。

如果你也曾从这些小小的创造里得到过快乐，大概能理解我为什么还愿意在此刻继续前行。

仅此而已。

以上为作者犯病

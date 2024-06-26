# RUC 学务活动检测器

## 简介
本系统适用于 `windows` 下对中国人民大学为人大平台学务中心上的活动进行监视，并自动对新活动进行注册报名的行为，旨在帮助校友进行自动监察新活动并进行报名。

## 安装
首先，用户应当克隆项目到本地。
```bash
git clone git@github.com:SummerFall1819/RenminSpider.git
```

之后，项目应当安装必要的依赖，其中必须使用的依赖可以以如下命令安装。
```bash
cd RUCSpider
pip install -r requirements.txt
```

**注意**：为了识别中国人民大学登陆时必要的验证码信息，您需要以下三项之一：
1. 程序每次进行登录时，手动输入验证码信息，此方法无需其他依赖。您需要在启动项目前，将 `setting.yml` 文件中 `manual` 改为 `True`，程序会提供简洁的 `tkinker` GUI 等待输入。
2. 采取本仓库提供的 `ddddocr` 实现方式。额外的依赖已经在 `requirements.txt` 中列出并安装。
    - **注意**：由于 `ddddocr` 使用 `PIL.image` 中已经在 `10.0.0` 弃用的方法 `Image.ANTIALIAS`. 因此需要对 `Pillow` 降级以避免此错误。或者，修改 `ddddocr` 的 `_init_.py` 文件，将其中的 `ANTIALIAS` 替换为新方法 `LANCZOS`. 出于自动配置的需要，这里使用了 `Pillow` 降级的方式。
    对应错误：
        ```bash
        image = image.resize((int(image.size[0] * (64 / image.size[1])), 64), Image.ANTIALIAS).convert('L')
        ```
    - **注意**: 由于 `ddddocr` 不能一定正确识别验证码，因此可能会导致产生其他错误，该错误是由于识别失败导致无法获取 token 信息，程序将产生 `error.txt` 文件和一个 `png` 形式的验证码文件以供校对。一旦发生此类错误，程序将立刻停止。如果由于错误的学号和密码进行多次尝试可能会冻结账号，**请确保学号和密码正确**。
        ```bash
        Unexpected error. Check username, password and captcha in 'error.txt' to ensure the login process is correct.
        ```


3. 自行配置 OCR 环境，并在 `utils.py` 中重写函数。
    您可以重写 `OCRCODE` 函数，该函数接受以 `base64` 编码的 `png` 图片，最终返回该验证码对应的文本。

以上步骤完成后，您的项目文件树应当如下所示。
```apache
RUCSpider
│  alias.json
│  main.py
│  readme.md
│  requirements.txt
│  schedule.yml
│  setting.yml
│  spiderexcep.py
│  utils.py
```

## 使用
在正式使用之前，应当先配置 `setting.yml` 文件。
其中包含的内容如下：
- `cookies` 程序会使用 `cookie` 直接登录微人大。初次使用无需填写任何信息，程序会自动抓取。
- `expire_time`: 记录 `cookie` 的有效期限，并确保每次重启动时更新。
- `interval_seconds`: 记录监视微人大学务的间隔时间，按秒记录。
- `password`: **必填**， 用户的登陆密码。形式为 `password: xxxxxxxxx`
    **此程序已完全开源，并保证本地部署不会外传任何敏感信息，但用户在使用程序时仍应当注意防止泄露密码。** 
- `username`: **必填**， 用户的学号。形式为 `username: 20xxxxxxxx`
- `manual`: 是否使用手动输入验证码，如果在安装时决定采用方法 1, 请修改为 `True`.
- `notifier`: 提醒渠道。从`box_alert`（弹窗）、`wx_alert`（微信推送）或者`none`（不提醒）中选择
- `wxpusher_appToken`: 在[管理后台WxPusher](https://wxpusher.zjiecode.com/admin)创建一个私有App，并填入对应Token
- `wxpusher_uid`: [关注公众号](https://wxpusher.zjiecode.com/demo)并获取微信帐号对应的UID

在完成修改之后，(激活虚拟环境)，运行 `main.py` 即可。

使用者应当注意，对于提示“报名成功”的活动，并不一定具有参与资格。活动可能取消或延期，请及时登陆微人大查看参会信息。

**短期行为**: 新增了弹窗提醒功能，目前版本请在 `RUCSpider()` 初始化时指定 `window_alert = True` 启用该功能，理论上弹窗适配各平台，但未经测试。也可以指定 `window_icon = <path>` 来修改自己的弹窗图标。当前版本默认启动，之后重构时会放入 `setting.yml` 中。

## 其他细节
### 基于本程序的其他改进

#### 筛选器
本程序提供了一个 `filter` 接口，以协助进行必要的讲座条件检查。默认不进行筛选，但仍旧实现了一个简单的筛选器：其根据你给出的空闲时间区间检查讲座是否在空闲时间区间内。如果用户希望使用这个筛选器，应当修改 `schedule.yml` 下列出的七天的信息。阅读 `yaml` 文件在此不作赘述。时间从星期一到星期天。修改完 `schedule.yml` 后，在 `main.py` 下 `schedule.every(INTERVAL).seconds.do` 函数处增加参数 `filter = filt` 即可。

如果希望自定义一个筛选器，其筛选器函数要求如下：
```python
def func(lec:dict):
    """
    This function take excatly one lecture information, and check if it satisfy the restrictions.

    Args:
        lec (dict): lecture information.
    Returns:
        bool: True if the lecture satisfy the restrictions, false otherwise.
    """
    
    # your code here.
```

你可以通过 `python` 的柯里化引入更多参数。

#### OCR 接入
除开重写 `OCR` 之外，你也可以定义自己的方式来获取验证码，并在 `main.py`, `RUCSpider` 初始化处加入 `captcha_func = [Callable]` 加入函数要求如下
```python
def captcha_func(base_64_img:str|bytes):
    """
    This function load in a base64 encoded image, and return the captcha code.
    
    Args:
        base_64_img (str|bytes): base64 encoded image.
    
    Returns:
        str: captcha code.
    """

    # Your code here.
```
这种函数是不限于 `OCR` 方式的，你可以自主设计任何的方式，只要能从中获得验证码即可。

#### 讲座类型筛选
本程序默认是处于筛选形势与政策讲座而设计的，在 `alias.json` 下保存了少量的条件筛选内容。如果您希望进行其他讲座的监看，请修改 `main.py` 下 `CONDITION`, 其元素数目为三个，分别对应活动大类、活动小类、小类子类的筛选。
(活动状态和组织单位的实现与监看的初衷冲突，故未实现)，如果希望查找活动名称，那么在 `PullLecture` 下加入 `Query` 为关键词即可。
**注意**：有些小类子类名称与活动小类冲突，这种情况请将小类子类更改为 “不限”，即可实现内容。


### 程序实现细节
@TODO 待完成吧。

## 其他
本程序尚未完全完成，但目前可以实现基本功能。在后期会逐步改进。

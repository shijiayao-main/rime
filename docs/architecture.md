# Rime 墨奇码配置架构分析

> 本文档分析本仓库（基于 [rime-shuangpin-fuzhuma](https://github.com/gaboolic/rime-shuangpin-fuzhuma)）的整体实现方式与多方案支持原理。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────┐
│                   librime 引擎核心                    │
│  （C++ 编写，跨平台，处理所有输入法底层逻辑）           │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐│
│  │  Processor  │  │ Segmentor  │  │   Translator   ││
│  │ （按键处理） │  │（分段识别）│  │  （编码→候选）  ││
│  └────────────┘  └────────────┘  └────────────────┘│
│                                                     │
│  ┌────────────┐  ┌────────────────────────────────┐ │
│  │   Filter   │  │        Grammar/Language Model   │ │
│  │（候选过滤） │  │         （语言模型打分）          │ │
│  └────────────┘  └────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
         ↑
         │ 读取 YAML 配置 + 执行 Lua 脚本
         │
┌─────────────────────────────────────────────────────┐
│                本仓库配置文件层                        │
│                                                     │
│  default.yaml       → 全局入口（方案列表）             │
│  moqi.yaml          → 公共配置模板（共享组件）          │
│  moqi_speller.yaml  → 拼写转换规则模板                 │
│  *.schema.yaml      → 各输入方案定义                   │
│  *.dict.yaml        → 字/词典数据                      │
│  lua/               → Lua 扩展脚本                    │
└─────────────────────────────────────────────────────┘
```

---

## 二、多方案支持原理

### 2.1 方案注册：`default.yaml`

`default.yaml` 是整个配置的入口，通过 `schema_list` 注册所有可用方案：

```yaml
schema_list:
  - schema: moqi_wan_flypymo   # 墨奇·小鹤+墨奇形（推荐）
  - schema: moqi_wan_zrm       # 墨奇·自然码+自然码部首辅助码
  - schema: moqi_wan_flypy     # 墨奇·小鹤+鹤形
  - schema: moqi_single_xh     # 墨奇码·顶屏版·小鹤双拼
  # ... 更多方案（注释掉即为禁用）
```

每个 `schema` 对应一个 `*.schema.yaml` 文件，这是 Rime 识别方案的约定。

---

### 2.2 核心设计：`moqi.yaml` 公共模板

这是本配置最关键的设计——**把所有方案共用的组件抽离到 `moqi.yaml`**，各方案通过 `__include` 指令引用，避免重复配置。

**`moqi.yaml` 包含的共享组件：**

| 键名 | 用途 |
|------|------|
| `switches_engine` | 开关列表（简繁/全半角/Emoji等）+ 引擎处理器/分段器/翻译器/过滤器完整声明 |
| `phrase` | 所有置顶词典（custom_phrase、快符、字根、tab简码等）配置 |
| `reverse` | 所有反查子方案（墨奇反查、部件组字、仓颉、笔画、两分等） |
| `opencc_config` | OpenCC 简繁转换、拆字显示、火星文等 simplifier 配置 |
| `punctuator` | 标点符号映射规则 |
| `guide` | 引导前缀（日期、Unicode、计算器等功能的触发前缀） |
| `big_char_and_user_dict` | 大字集翻译器、用户词典翻译器 |
| `grammar` | 语言模型配置 |

**各方案的引用方式：**

```yaml
# moqi_wan_flypymo.schema.yaml
__include: moqi.yaml:/switches_engine
__include: moqi.yaml:/phrase
__include: moqi.yaml:/reverse
__include: moqi.yaml:/opencc_config
__include: moqi.yaml:/punctuator
__include: moqi.yaml:/guide
```

`__include: 文件名:/键名` 是 librime 提供的 YAML 片段引用机制，实现了配置的模块化复用。

---

### 2.3 双拼方案差异：`moqi_speller.yaml` 拼写规则模板

不同双拼方案（小鹤、自然码、全拼等）之间的区别，**几乎完全体现在 `speller/algebra` 中的拼写转换规则**。

`moqi_speller.yaml` 将各方案的拼写转换规则分别定义为独立的 YAML 列表节点：

```yaml
flypy_speller:   # 小鹤双拼转换规则
  - derive/^([jqxy])u(;.*)$/$1v$2/
  - xform/^(\w+?)iu(;.*)/$1Ⓠ$2/
  # ...

zrm_speller:     # 自然码双拼转换规则
  - derive/^([jqxy])u(;.*)$/$1v$2/
  - xform/^(\w+?)iu(;.*)$/$1Ⓠ$2/
  # ...（映射规则不同）

quanpin_speller: # 全拼规则
  - erase/^xx;.*$/
```

各方案在自己的 `schema.yaml` 中选择引用对应规则：

```yaml
# moqi_wan_flypymo.schema.yaml（小鹤双拼+墨奇形）
speller:
  algebra:
    __include: moqi_speller.yaml:/flypy_speller

# moqi_wan_zrm.schema.yaml（自然码双拼+自然码辅助码）
speller:
  algebra:
    __include: moqi_speller.yaml:/zrm_speller
```

**辅助码差异**通过 `__patch` + `aux_patch` 节点追加到 algebra 后面：

```yaml
aux_patch:
  __include: moqi_speller.yaml:/moqi_aux   # 墨奇形辅助码规则
  __append:
    __include: moqi_speller.yaml:/common_aux

__patch:
  speller/algebra/+:
    __include: aux_patch
```

---

### 2.4 词典架构：统一大词典 + 模块化导入

所有方案**共用同一份词典** `moqi_wan.extended`，通过 `moqi_wan.extended.dict.yaml` 的 `import_tables` 机制模块化组合：

```yaml
# moqi_wan.extended.dict.yaml
name: moqi_wan.extended
import_tables:
  - cn_dicts/8105       # 8105 常用字表（含多列辅助码编码）
  - cn_dicts/41448      # 4万字大字表
  - cn_dicts/base       # 基础词库
  - cn_dicts/ext        # 扩展词库
  - cn_dicts_common/user  # 用户自定义词典
  - cn_dicts_cell/idiom   # 细胞词库：成语
  - cn_dicts_cell/food    # 细胞词库：食物
  # ... 按需启用/禁用各细胞词库
```

词典条目格式为：`汉字\t拼音;墨奇辅助;鹤形辅助;自然码辅助;...`（多列辅助码），`speller/algebra` 的转换规则负责从同一份数据中提取出当前方案需要的编码列。

---

## 三、YAML 配置指令机制

librime 支持以下特殊 YAML 指令，是本配置实现模块化的基础：

| 指令 | 作用 |
|------|------|
| `__include: file.yaml:/key` | 将指定文件的指定节点内容原地展开合并 |
| `__patch:` | 对已有配置进行局部覆盖/修改 |
| `__append:` | 向列表末尾追加条目 |
| `import_preset: default` | 继承 `default.yaml` 中同名节点的设置 |

这套指令系统使得：
- **公共配置只写一次**（在 `moqi.yaml`），各方案通过 `__include` 引用
- **方案差异只声明差异部分**，用 `__patch` 覆盖默认值
- **词典可按需组合**，通过 `import_tables` 控制哪些词库生效

---

## 四、Lua 扩展脚本机制

Rime 通过 `librime-lua` 插件支持 Lua 脚本，扩展了三类组件：

### 4.1 注册方式

在 `moqi.yaml:/switches_engine` 的 `engine` 中声明：

```yaml
engine:
  processors:
    - lua_processor@*sbxlm.key_binder   # Lua 处理器
  translators:
    - lua_translator@*date_translator   # Lua 翻译器
    - lua_translator@*lunar
    - lua_translator@*unicode
    - lua_translator@*calculator
  filters:
    - lua_filter@*pro_comment_format    # Lua 过滤器
    - lua_filter@*aux_lookup_filter
    - lua_filter@*stick
    - lua_filter@*is_in_user_dict
    - lua_filter@*moqi.jianma_show
```

`@*模块名` 对应 `lua/模块名.lua` 文件；`@*moqi.jianma_show` 对应 `lua/moqi/jianma_show.lua`（子目录）。

### 4.2 各 Lua 脚本功能说明

| 脚本 | 类型 | 功能 |
|------|------|------|
| `date_translator.lua` | 翻译器 | 输入 `rq/sj/xq/dt/ts` 输出日期/时间/星期/ISO时间/时间戳 |
| `lunar.lua` | 翻译器 | 农历日期输出 |
| `unicode.lua` | 翻译器 | Unicode 码位输入（`U+XXXX`） |
| `calculator.lua` | 翻译器 | 计算器功能 |
| `number_translator.lua` | 翻译器 | 数字/金额大写转换 |
| `card_number.lua` | 翻译器 | 银行卡号格式化 |
| `pro_comment_format.lua` | 过滤器 | 超级注释模块：辅助码提醒 + 错音错字提示 |
| `aux_lookup_filter.lua` | 过滤器 | `` ` `` 引导辅助码实时筛选候选 |
| `stick.lua` | 过滤器 | 简码词库回显（Tab上屏词置顶 + ⚡标记） |
| `is_in_user_dict.lua` | 过滤器 | 用户词典词条加 `*` 标记 |
| `moqi/jianma_show.lua` | 过滤器 | 全码输入时回显简码提示 |
| `sbxlm/key_binder.lua` | 处理器 | 自定义按键绑定 |
| `pin_cand_filter.lua` | 过滤器 | 候选项固定置顶 |
| `long_word_filter.lua` | 过滤器 | 长词优先排序 |

### 4.3 Lua 脚本与配置的交互

Lua 脚本通过 `env.engine.schema.config` 读取方案中的 YAML 配置项：

```lua
-- 以 pro_comment_format.lua 为例
local config = env.engine.schema.config
local fuzhu_type = config:get_string("pro_comment_format/fuzhu_type")
local fuzhu_code = config:get_bool("pro_comment_format/fuzhu_code")
```

这样每个方案只需在自己的 `schema.yaml` 顶层声明对应参数，就能控制 Lua 脚本的行为：

```yaml
# moqi_wan_flypymo.schema.yaml
pro_comment_format:
  fuzhu_code: true
  fuzhu_type: moqi      # ← 控制显示墨奇形辅助码

# moqi_wan_zrm.schema.yaml  
pro_comment_format:
  fuzhu_code: true
  fuzhu_type: zrm       # ← 控制显示自然码辅助码
```

---

## 五、反查方案机制

通过 `affix_segmentor` + `table_translator` 实现前缀引导的反查：

| 前缀 | 反查方案 | 词典 |
|------|----------|------|
| `amq` | 墨奇码反查 | `reverse_moqima` |
| `az` | 部件组字反查 | `radical_flypy` |
| `alf` | 自然两分反查 | `zrlf` |
| `arj` | 仓颉五代反查 | `cangjie5` |
| 笔画前缀 | 笔画反查 | `stroke` |

反查配置同样抽离到 `moqi.yaml:/reverse`，各方案通过 `__include` 统一引用。

---

## 六、前端适配

| 文件 | 对应前端 |
|------|----------|
| `squirrel.yaml` | macOS 鼠鬚管（Squirrel） |
| `fcitx5.yaml` | Linux fcitx5-rime |
| `简纯.trime.yaml` | Android 同文输入法（Trime） |

各前端的外观、主题、候选栏方向等 UI 配置独立存放，引擎层配置（`*.schema.yaml`、`*.dict.yaml`、`lua/`）对所有前端通用。

---

## 七、总结：方案差异对比

| 方案文件 | 双拼方案 | 辅助码 | speller 引用 |
|----------|----------|--------|--------------|
| `moqi_wan_flypymo` | 小鹤双拼 | 墨奇形 | `flypy_speller` + `moqi_aux` |
| `moqi_wan_flypy` | 小鹤双拼 | 鹤形 | `flypy_speller` + `flypy_aux` |
| `moqi_wan_zrm` | 自然码双拼 | 自然码部首 | `zrm_speller` + `zrm_aux` |
| `moqi_wan_flypyhu` | 小鹤双拼 | 虎形 | `flypy_speller` + `tiger_aux` |
| `moqi_wan_jdh` | 小鹤双拼 | 简单鹤 | `flypy_speller` + `jdh_aux` |
| `moqi_single_xh` | 小鹤双拼 | （顶屏4码自动上屏） | 独立配置 |

所有方案**共享**：
- 同一份大词典（`moqi_wan.extended`）
- 同一套引擎处理流程（来自 `moqi.yaml:/switches_engine`）
- 同一套 Lua 功能脚本（lua/ 目录下）
- 同一套反查方案（来自 `moqi.yaml:/reverse`）

**只有 `speller/algebra` 中的编码转换规则和 `pro_comment_format/fuzhu_type` 因方案而异**，这正是整套配置能以极低的重复度支持多种输入方案的核心设计。

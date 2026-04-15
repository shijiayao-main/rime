# README
来自墨奇码 https://github.com/gaboolic/rime-shuangpin-fuzhuma

## 注意事项

### 字体安装

macos
```shell
# 1. Noto CJK 全覆盖汉字（简繁日韩 99.9%）
brew install --cask font-noto-sans-cjk-sc font-noto-serif-cjk-sc

# 2. 花园明朝 HanaMin（超罕见字、古文、方言字）
brew install --cask font-hanazono-mincho

# 3. Noto 彩色 Emoji（解决黑白方框）
brew install --cask font-noto-color-emoji

# 4. 刷新字体缓存
atsutil databases -removeUser
```

linux
```shell
# 1. 安装 Noto CJK（覆盖99.9%常用/生僻汉字、简繁日韩）
sudo apt update
sudo apt install -y fonts-noto-cjk fonts-noto-cjk-extra

# 2. 安装花园明朝（HanaMin，补全超罕见Unicode汉字、古文/方言字）
sudo apt install -y fonts-hanazono

# 3. 安装彩色Emoji（解决Rime输入表情显示黑白/方框）
sudo apt install -y fonts-noto-color-emoji

# 4. 刷新字体缓存
fc-cache -fv
```

### linux下别用flatpak了不好用自己编译很简单

项目名称	核心用途	HTTPS 克隆地址	SSH 克隆地址	仓库主页
librime 输入法引擎核心	Rime 跨平台核心引擎本体，所有前端的底层依赖，编译安装核心对象	https://github.com/rime/librime.git	git@github.com:rime/librime.git	https://github.com/rime/librime
librime-lua 扩展插件	Rime 官方认可的 Lua 扩展插件，支持 Lua 脚本自定义输入逻辑、翻译器、过滤器、处理器	https://github.com/hchunhui/librime-lua.git	git@github.com:hchunhui/librime-lua.git	https://github.com/hchunhui/librime-lua


``` shell
cd ~/Workspace/Github/librime/

# 1. 清理 librime-lua 独立编译的 build 目录
rm -rf librime-lua/build/

# 2. 清理 librime 主目录之前的编译缓存（确保从零开始）
rm -rf build/

# 3. 【关键】确保 librime-lua 在 plugins 目录下（librime 官方插件机制要求）
# 如果不在，移动过去：
mv -f librime-lua plugins/
```

``` shell
# 创建构建目录
mkdir build && cd build

# CMake 配置（自动识别 plugins 目录下的 librime-lua，自动处理所有依赖）
cmake .. -DCMAKE_BUILD_TYPE=Release -DBUILD_TEST=OFF -DBUILD_SHARED_LIBS=ON

# 编译（根据 CPU 核心数自动并行）
make -j$(nproc)

# 安装到系统
sudo make install

# 更新动态链接库缓存（关键！否则 fcitx5 找不到新库）
sudo ldconfig
```


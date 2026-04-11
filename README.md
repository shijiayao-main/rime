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

### linux下使用flatpak能减少绝大多数的问题
https://blog.iamcheyan.com/2025-11-03-%E4%BD%BF%E7%94%A8Flatpak%E5%AE%89%E8%A3%85Fcitx5%E5%92%8CRime%E8%BE%93%E5%85%A5%E6%B3%95.html

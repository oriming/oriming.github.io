---
title: 鼠须管(Squirrel) color_scheme 配置
author: Ori Ming
date: 2025-05-17 8:00:00 +0800
categories: [Squirrel]
tags: [Squirrel]
render_with_liquid: false
---

# Welcome

本文主要做笔记，记录鼠须管好看的主题。

演示环境:

+ macOS Sequoia 15.5
+ Squirrel 1.0.3

## 1. 配置主题

1. 打开配置文件
![配置文件地址](/assets/img/2025-05-17-squirrel/2025-05-17-07-59-11.png)

2. 找到主题标签 `style.color_scheme` 和 `style.color_scheme_dark`，并设定主题风格

```yaml
style:
  # 选择皮肤，亮色与暗色主题
  # color_scheme: onctf
  color_scheme: wechat
  color_scheme_dark: wechat_dark
```

## 2.我收藏的主题

配置位置：在 `squirrel.yaml` 文件末尾找到 `preset_color_schemes` 标签下粘贴一下主题。

```yaml
# 皮肤列表
preset_color_schemes:

  native:
    name: 系統配色

  onctf:
    name: '精简风'
    author: '骁隆'
    back_color: 0xFFFFFF
    border_height: 0
    border_width: 8
    candidate_format: '%c %@ '
    comment_text_color: 0x999999
    corner_radius: 5
    hilited_corner_radius: 5
    font_face: "PingFangSC"
    font_point: 16
    hilited_candidate_back_color: 0x444444
    hilited_candidate_text_color: 0xFFFFFF
    horizontal: true
    inline_preedit: true
    label_font_point: 12
    text_color: 0x424242

  mint_dark_green:
    name: "碧月青／Mint Dark Green"
    author: Mintimate <"Mintimate's Blog">
    translucency: true                      # 磨砂： false | true
    mutual_exclusive: false                 # 色不叠加： false | true
    color_space: srgb
    back_color: 0x323232                    # 底色
    candidate_text_color: 0xE8E8E8          # 文字颜色
    comment_text_color: 0xBEBEBE            # 注颜色
    label_color: 0xB0B0B0                   # 序号颜色
    hilited_candidate_back_color: 0x83C81C  # 选中底色
    hilited_candidate_text_color: 0xEFEFEF  # 选中文字颜色
    hilited_comment_text_color: 0xF4FAF8    # 选中注颜色
    hilited_candidate_label_color: 0xEFEFEF # 选中序号颜色
    text_color: 0x83C81C                    # 拼音颜色 （inline_preedit: false）
    hilited_text_color: 0xed9564            # 选中拼音颜色 （inline_preedit: false）

  reimu:
    name: "灵梦／Reimu"
    author: "Lufs X <i@isteed.cc>"
    font_face: "LXGWWenKai-Regular, PingFangSC"
    font_point: 17
    label_font_face: "LXGWWenKai-Regular, PingFangSC"
    label_font_point: 14
    candidate_format: "[label]\u2005[candidate] [comment]"
    candidate_list_layout: linear
    text_orientation: horizontal
    inline_preedit: true
    corner_radius: 7
    hilited_corner_radius: 6
    border_height: 1
    border_width: 1
    alpha: 0.95
    shadow_size: 2
    color_space: display_p3
    back_color: 0xF5FCFD
    candidate_text_color: 0x282C32
    comment_text_color: 0x717172
    label_color: 0x888785
    hilited_candidate_back_color: 0xF5FCFD
    hilited_candidate_text_color: 0x4F00E5
    hilited_comment_text_color: 0x9F9CF2
    hilited_candidate_label_color: 0x4F00E5
    text_color: 0x6B54E9
    hilited_text_color: 0xD8000000

  reimu_dark:
    name: "灵梦／Reimu／深色"
    author: "Lufs X <i@isteed.cc>"
    font_face: "LXGWWenKai-Regular, PingFangSC"
    font_point: 17
    label_font_face: "LXGWWenKai-Regular, PingFangSC"
    label_font_point: 14
    candidate_format: "[label]\u2005[candidate] [comment]"
    candidate_list_layout: linear
    text_orientation: horizontal
    inline_preedit: true
    corner_radius: 7
    hilited_corner_radius: 6
    border_height: 1
    border_width: 1
    alpha: 0.95
    shadow_size: 2
    color_space: display_p3
    back_color: 0x020A00
    border_color: 0x020A00
    candidate_text_color: 0xC0C0C0
    comment_text_color: 0x717172
    label_color: 0x717172
    hilited_candidate_back_color: 0x0C140A
    hilited_candidate_text_color: 0x3100C7
    hilited_comment_text_color: 0x7772AF
    hilited_candidate_label_color: 0x3100C7
    text_color: 0x6B54E9
    hilited_text_color: 0xD8000000

  wechat:
    name: "高仿微信输入法"
    author: "Lufs X <i@isteed.cc>"
    font_face: "PingFangSC-Regular"
    font_point: 17
    label_font_face: "PingFangSC-Regular"
    label_font_point: 14
    comment_font_face: "PingFangSC-Regular"
    comment_font_point: 14
    candidate_format: "[label]\u2005[candidate] [comment]"
    candidate_list_layout: linear
    text_orientation: horizontal
    inline_preedit: true
    corner_radius: 7
    hilited_corner_radius: 7
    border_height: -2
    color_space: display_p3
    back_color: 0xFFFFFF
    border_color: 0xFFFFFF
    candidate_text_color: 0x444444
    comment_text_color: 0x8E8E8E
    label_color: 0x888785
    hilited_candidate_back_color: 0x7BAE4F
    hilited_candidate_text_color: 0xFFFFFF
    hilited_comment_text_color: 0xF0F0F0
    hilited_candidate_label_color: 0xFFFFFF
    text_color: 0xFFFFFF
    hilited_text_color: 0xD8000000

  wechat_dark:
    name: "高仿暗色微信输入法"
    author: "Lufs X <i@isteed.cc>"
    font_face: "PingFangSC-Regular"
    font_point: 17
    label_font_face: "PingFangSC-Regular"
    label_font_point: 14
    comment_font_face: "PingFangSC-Regular"
    comment_font_point: 14
    candidate_format: "[label]\u2005[candidate] [comment]"
    candidate_list_layout: linear
    text_orientation: horizontal
    inline_preedit: true
    corner_radius: 7
    hilited_corner_radius: 7
    border_height: -2
    color_space: display_p3
    back_color: 0x151515
    border_color: 0x151515
    candidate_text_color: 0xB9B9B9
    comment_text_color: 0x8E8E8E
    label_color: 0x888785
    hilited_candidate_back_color: 0x74A54B
    hilited_candidate_text_color: 0xFFFFFF
    hilited_comment_text_color: 0xF0F0F0
    hilited_candidate_label_color: 0xFFFFFF
    text_color: 0xFFFFFF
    hilited_text_color: 0x777777
```

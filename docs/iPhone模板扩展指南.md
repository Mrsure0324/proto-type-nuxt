# iPhone样机模板扩展指南

## 概述

iPhone样机模板定义在 `pages/mockup/index.vue` 文件中的 `protoTypeList` 数组中。每个模板包含预览图、样机配置列表和背景设置。

## 模板结构

### 基本结构

```javascript
{
  defaultProtoUrl: '/image/proto/your-preview-image.png',  // 模板预览图路径
  exampleName: t("mockup.yourTemplateName"),              // 模板名称（需要添加到i18n）
  protoList: [                                             // 样机配置数组（可包含多个iPhone）
    {
      type: 'iphoneType',                                  // 设备类型，固定为 'iphoneType'
      name: 'iPhone',                                      // 样机名称（用于标识，可自定义）
      frame: true,                                         // 是否显示边框（true/false）
      smartIsLand: true,                                   // 是否显示灵动岛（true/false）
      style: {                                             // 样机样式（位置、旋转、缩放等）
        width: '426px',
        height: '877px',
        transform: 'rotate(0deg) translateX(0px) translateY(0px) scaleX(1.0) scaleY(1.0) ...',
        filter: 'drop-shadow(...)'                        // 可选：阴影效果
      },
      paperStyleMethod: (proto) => {                       // 壁纸样式方法
        return "width: calc(100% - 36px);height: calc(100% - 32px); border-radius: 50px;position: absolute;left: 18px;top: 16px;"
      },
      paperChatStyle: {                                    // 聊天界面样式（如果使用聊天模式）
        position: 'absolute',
        width: '390px',
        height: '653.12px',
        top: '115px',
        left: '18px',
      },
      screenType: screenOptions['iphoneType'][0],          // 屏幕类型：0=锁屏, 1=桌面, 2=仅样机
      selectedTime: new Date(),                            // 默认时间
      selectedDate: new Date(),                            // 默认日期
      systemColor: "#ffffff",                              // 系统颜色
      dateTimeColor: "#ffffff"                             // 日期时间颜色
    }
  ],
  background: {                                            // 背景配置
    type: backgroundOptions[0],                            // 背景类型
    needBlur: true,                                        // 是否需要模糊效果
    autoUpdate: true,                                      // 是否自动更新背景
    defaultBackgroundUrl: '/image/background/your-bg.jpg', // 默认背景图（可选）
    backgroundImageStyle: () => {                          // 背景图片样式方法
      return {
        display: !selectedProto.value.background.needBlur ? 'block' : 'none',
      }
    },
  }
}
```

## 添加新模板的步骤

### 步骤1：准备预览图

将模板预览图放到 `public/image/proto/` 目录下，例如：
- `public/image/proto/my-new-template.png`

### 步骤2：准备背景图（可选）

如果需要自定义背景，将背景图放到 `public/image/background/` 目录下。

### 步骤3：在代码中添加图片引用

在 `pages/mockup/index.vue` 文件顶部（约第25-54行）添加图片引用：

```javascript
const myNewTemplateImage = '/image/proto/my-new-template.png'
const myNewTemplateBackgroundImage = '/image/background/my-new-template-bg.jpg'
```

### 步骤4：添加模板配置

在 `protoTypeList` 数组中找到 `protoType: 'iPhone'` 的对象，在其 `list` 数组中添加新模板：

```javascript
{
  protoType: 'iPhone',
  list: [
    // ... 现有模板 ...
    {
      defaultProtoUrl: myNewTemplateImage,
      exampleName: t("mockup.myNewTemplate"),
      protoList: [
        {
          type: 'iphoneType',
          name: 'iPhone',
          frame: true,
          smartIsLand: true,
          style: {
            width: '426px',
            height: '877px',
            transform: 'rotate(0deg) translateX(0px) translateY(0px) scaleX(1.0) scaleY(1.0) perspective(none) skewX(0deg) skewY(0deg) rotateX(0deg) rotateY(0deg) rotateZ(0deg)',
            // 可选：添加阴影
            // filter: 'drop-shadow(rgba(0, 0, 0, 0.25) -10px 10px 10px)',
          },
          paperStyleMethod: (proto) => {
            return "width: calc(100% - 36px);height: calc(100% - 32px); border-radius: 50px;position: absolute;left: 18px;top: 16px;"
          },
          paperChatStyle: {
            position: 'absolute',
            width: '390px',
            height: '653.12px',
            top: '115px',
            left: '18px',
          },
          screenType: screenOptions['iphoneType'][0],
          selectedTime: new Date(),
          selectedDate: new Date(),
          systemColor: defaultSystemColor,
          dateTimeColor: defaultDateTimeColor
        }
      ],
      background: {
        type: backgroundOptions[0],
        needBlur: true,
        autoUpdate: true,
        backgroundImageStyle: () => {
          return {
            display: !selectedProto.value.background.needBlur ? 'block' : 'none',
          }
        },
      }
    }
  ]
}
```

### 步骤5：添加国际化文本

在 `i18n/locales/zh.json` 和 `i18n/locales/en.json` 中添加模板名称：

**zh.json:**
```json
{
  "mockup": {
    "myNewTemplate": "我的新模板"
  }
}
```

**en.json:**
```json
{
  "mockup": {
    "myNewTemplate": "My New Template"
  }
}
```

## 样式参数详解

### transform 参数说明

`transform` 属性控制样机的位置、旋转和缩放：

```javascript
transform: 'rotate(角度deg) translateX(X偏移px) translateY(Y偏移px) scaleX(X缩放) scaleY(Y缩放) ...'
```

**示例：**
- `rotate(0deg)` - 不旋转
- `rotate(45deg)` - 旋转45度
- `translateX(100px)` - 向右移动100px
- `translateY(-50px)` - 向上移动50px
- `scaleX(1.5)` - X轴放大1.5倍
- `scaleY(1.5)` - Y轴放大1.5倍

### 多设备模板

如果模板需要包含多个iPhone，只需在 `protoList` 数组中添加多个对象：

```javascript
protoList: [
  {
    type: 'iphoneType',
    name: '左边iPhone',
    // ... 配置 ...
    style: {
      transform: 'rotate(0deg) translateX(-265px) translateY(0px) scaleX(1.0) scaleY(1.0) ...'
    }
  },
  {
    type: 'iphoneType',
    name: '右边iPhone',
    // ... 配置 ...
    style: {
      transform: 'rotate(0deg) translateX(265px) translateY(0px) scaleX(1.0) scaleY(1.0) ...'
    }
  }
]
```

## 常用模板示例

### 示例1：简单居中模板

```javascript
{
  defaultProtoUrl: simpleImage,
  exampleName: t("mockup.simple"),
  protoList: [{
    type: 'iphoneType',
    name: 'iPhone',
    frame: true,
    smartIsLand: true,
    style: {
      width: '426px',
      height: '877px',
      transform: 'rotate(0deg) translateX(0px) translateY(0px) scaleX(1.52793) scaleY(1.52794) perspective(none) skewX(0deg) skewY(0deg) rotateX(0deg) rotateY(0deg) rotateZ(0deg)',
    },
    paperStyleMethod: (proto) => {
      return "width: calc(100% - 36px);height: calc(100% - 32px); border-radius: 50px;position: absolute;left: 18px;top: 16px;"
    },
    paperChatStyle: {
      position: 'absolute',
      width: '390px',
      height: '653.12px',
      top: '115px',
      left: '18px',
    },
    screenType: screenOptions['iphoneType'][0],
    selectedTime: new Date(),
    selectedDate: new Date(),
    systemColor: defaultSystemColor,
    dateTimeColor: defaultDateTimeColor
  }],
  background: {
    type: backgroundOptions[0],
    needBlur: true,
    autoUpdate: true,
    backgroundImageStyle: () => ({
      display: !selectedProto.value.background.needBlur ? 'block' : 'none',
    }),
  }
}
```

### 示例2：带旋转和阴影的模板

```javascript
{
  defaultProtoUrl: rotatedImage,
  exampleName: t("mockup.rotated"),
  protoList: [{
    type: 'iphoneType',
    name: 'iPhone',
    frame: true,
    smartIsLand: true,
    style: {
      width: '426px',
      height: '877px',
      transform: 'rotate(10deg) translateX(50px) translateY(130px) scaleX(1.2) scaleY(1.2) perspective(none) skewX(0deg) skewY(0deg) rotateX(0deg) rotateY(0deg) rotateZ(0deg)',
      filter: 'drop-shadow(rgba(0, 0, 0, 0.25) -10px 10px 10px)',
    },
    // ... 其他配置相同 ...
  }],
  // ... 背景配置 ...
}
```

### 示例3：无边框模板

```javascript
{
  defaultProtoUrl: noFrameImage,
  exampleName: t("mockup.noFrame"),
  protoList: [{
    type: 'iphoneType',
    name: 'iPhone',
    frame: false,              // 无边框
    smartIsLand: false,        // 无灵动岛
    style: {
      width: '426px',
      height: '877px',
      transform: 'rotate(0deg) translateX(0px) translateY(0px) scaleX(1.52793) scaleY(1.52794) perspective(none) skewX(0deg) skewY(0deg) rotateX(0deg) rotateY(0deg) rotateZ(0deg)',
      filter: 'drop-shadow(rgba(0, 0, 0, 0.5) -10px 5px 20px)',
    },
    // ... 其他配置 ...
  }],
  // ... 背景配置 ...
}
```

## 调试技巧

1. **调整位置**：修改 `transform` 中的 `translateX` 和 `translateY` 值
2. **调整大小**：修改 `scaleX` 和 `scaleY` 值，或直接修改 `width` 和 `height`
3. **调整角度**：修改 `rotate` 值
4. **预览效果**：在浏览器中实时查看效果，调整参数直到满意

## 注意事项

1. **图片路径**：确保图片路径正确，使用 `/image/` 开头的绝对路径
2. **国际化**：记得在 `zh.json` 和 `en.json` 中添加对应的翻译
3. **样式一致性**：`paperStyleMethod` 和 `paperChatStyle` 通常保持标准值，除非有特殊需求
4. **性能**：如果模板包含多个设备，注意性能影响

## 完整示例

查看 `pages/mockup/index.vue` 文件中的现有模板（从第439行开始）作为参考。


# AGENTS.md — 地球昼夜交替 3D 演示

## 项目本质
单文件 HTML 3D 教学演示（`index.html`），无构建工具、无服务器依赖、零 npm 依赖。双击即可在浏览器运行。所有代码和纹理在同一个仓库中。

## 关键约束（必读）
- **`file://` 协议兼容**：必须能用浏览器直接双击 `index.html` 打开。因此禁用 Three.js 的 `TextureLoader`（file:// 下 CORS 失败）。纹理改用 `<img>` 标签 + `img.decode()` 异步加载。
- **纹理路径**：所有纹理位于 `textures/` 目录，通过 `<img src="textures/xxx">` 引用。本地加载无需服务器。
- **Three.js 版本**：r160，通过 CDN 的 importmap 加载（`cdn.jsdelivr.net/npm/three@0.160.0`）。改版本须同步改 CDN 链接。

## 启动方式
```
# 无需任何命令，直接双击 index.html 即可
# 如需查看 console，用浏览器打开后按 F12
```

## 场景层级架构（不应搞错）
```
scene
  └─ axisGroup (rotation.x = TILT=23.44°)  ← 地轴固定指向星空
       ├─ spinGroup (rotation.y = 每天自转)  ← 所有地球上的物体挂这里
       │    ├─ 地球网格 (ShaderMaterial)
       │    ├─ 赤道圈、极圈、经度刻度圈
       │    ├─ 城市标记 (dot + label sprite)
       │    └─ ...
       ├─ 地轴线 (Line, 从 axisGroup 原点出发)
       └─ ...
  └─ 太阳 (Mesh + Sprite glow, 在 scene 层级，不在 spinGroup 内)
```
重要：**太阳不随地球旋转**。城市光照判定基于世界坐标点积（`dot.getWorldPosition().normalize() · sunDir`），不使用局部坐标。

## 夏至配置（不可随意改动）
- `TILT = 23.44°`（地轴倾角）
- `SUN_POS = (7.34, 2.93, -1.27)` — 夏至太阳方向，晨昏线与北极圈 (66.5°N) 相切
- 初始 `spinGroup.rotation.y = 2.618`（150°），使中国（~120°E）面向摄像机

## 纹理加载模式
```html
<!-- HTML 中隐藏的 img 标签 -->
<img id="texEarthDay" src="textures/earth_atmos_2048.jpg" style="display:none">
```
```js
// JS 中加载方式：
loadImgEl('texEarthDay').then(img => {
  const t = new THREE.Texture(img);
  t.colorSpace = THREE.SRGBColorSpace;
  t.needsUpdate = true;
  uniforms.dayTex.value = t;
});
```
`loadImgEl` 用 `img.decode()` 确保图像解码完成。所有纹理加载在异步 IIFE 中，失败时有 fallback canvas 纹理。

## 城市管理系统
- **CITY_DB**：内置城市数据库（34 个中国省级行政中心 + 20+ 世界城市）
- **INITIAL_CITIES**：7 个初始预设城市（北京、上海、乌鲁木齐、东京、悉尼、伦敦、纽约）
- **动态增删**：`addCityToScene(data)` / `removeCityById(id)` / `toggleCityVisibility(id)`
- **搜索**：基于 CITY_DB 的本地字符串匹配（`name.includes(q)`），不使用外部 API
- **ID 系统**：每个城市有唯一自增 `id`，非数组索引查找。面板元素 id 格式 `ci-${id}`、`cd-${id}`、`ct-${id}`
- **城市面板默认折叠**：点击标题展开/收起，展开时城市标记出现在地图上，收起时全部隐藏
- **防重复**：`addCityToScene` 检查 `city.name + city.country` 组合

## 命令速查
无构建/测试命令。修改后直接刷新浏览器验证。

## 开发注意事项
1. **不改纹理加载方式** — 不要试图换成 TextureLoader，会破坏 file:// 兼容
2. **不改层级结构** — axisGroup→spinGroup 的父子关系支撑了地轴固定逻辑和极昼模拟
3. **不改夏至常数** — TILT 和 SUN_POS 是教学正确性的基础
4. **城市操作后必须同步 Three.js 对象和 DOM** — 移除城市时要 dispose geometry/material 并从 spinGroup 移除
5. **城市标记放在 spinGroup 下** — 随地球自转，但点击聚焦时用 `getWorldPosition` 获取世界坐标
6. **动画循环每帧更新** — `updateCities()` 每 200ms 计算一次日照状态，cityData.forEach 处理可见性呼吸动画
7. **Shader 使用 `texture` 而非 `texture2D`** — WebGL2 兼容

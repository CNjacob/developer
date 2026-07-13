# UIKit UIView 与 CALayer 渲染机制

## 1. UIView 与 CALayer 的关系

`UIView` 是 UIKit 中负责界面组织、事件响应、布局管理的对象；`CALayer` 是 Core Animation 中负责图形内容、动画和合成的对象。

每个 `UIView` 默认都有一个根 `CALayer`：

```objc
UIView *view = [[UIView alloc] init];
CALayer *layer = view.layer;
```

职责划分：

| 对象 | 主要职责 |
| --- | --- |
| `UIView` | 事件响应、布局、层级管理、生命周期、Auto Layout |
| `CALayer` | 内容渲染、几何属性、阴影、圆角、动画、合成 |

```text
UIView
  ├── 处理触摸事件
  ├── 参与响应者链
  ├── 管理 subviews
  ├── 执行 layoutSubviews
  └── 持有 CALayer

CALayer
  ├── 保存 contents
  ├── 管理 sublayers
  ├── 提供动画能力
  └── 提交给 Render Server 合成
```

---

## 2. 视图层级与图层层级

### 2.1 View Tree

View Tree 是 UIKit 层的树：

```text
UIWindow
  └── RootView
      ├── HeaderView
      ├── TableView
      └── FooterView
```

它决定：

- 事件分发
- 布局传递
- 视图生命周期
- 子视图裁剪关系
- 坐标转换

### 2.2 Layer Tree

每个 View 对应一个 Layer，因此通常存在一棵 Layer Tree：

```text
Window Layer
  └── Root Layer
      ├── Header Layer
      ├── Table Layer
      └── Footer Layer
```

Layer Tree 决定最终要提交给 Core Animation 的可视内容。

### 2.3 三棵树模型

Core Animation 中常见三棵树：

| 树 | 作用 |
| --- | --- |
| Model Tree | App 操作的 layer 状态 |
| Presentation Tree | 屏幕上当前显示的动画中间状态 |
| Render Tree | 提交给渲染服务的内部表示 |

动画过程中：

```objc
layer.position = CGPointMake(200, 200);
```

Model Layer 已经变成目标值，而 Presentation Layer 表示屏幕当前正在显示的中间值。

---

## 3. UIKit 渲染管线

### 3.1 整体流程

```text
业务代码修改 UI
  ↓
UIView 标记 layout/display
  ↓
RunLoop 即将休眠
  ↓
Core Animation 提交事务
  ↓
CPU 计算布局、绘制 backing store
  ↓
Render Server 接收 layer tree
  ↓
GPU 合成
  ↓
屏幕显示
```

### 3.2 CPU 阶段

CPU 负责：

- 计算 Auto Layout
- 执行 `layoutSubviews`
- 文本排版
- 图片解码
- 执行 `drawRect:`
- 生成或更新 backing store
- 提交 Core Animation transaction

CPU 过重会导致主线程卡顿，常见原因包括：

- 复杂 Auto Layout
- 大量文本同步排版
- 主线程图片解码
- `drawRect:` 过重
- Cell 频繁创建和布局

### 3.3 GPU 阶段

GPU 负责：

- 图层纹理合成
- 透明混合
- 变换
- 圆角、mask、阴影等效果
- 最终 framebuffer 输出

GPU 过重常见原因：

- 图层层级过深
- 大面积透明混合
- 离屏渲染
- 超大图片纹理
- 复杂 mask 和动态阴影

---

## 4. 布局机制

### 4.1 Frame 布局

`frame` 是 View 在父视图坐标系中的位置和大小：

```objc
view.frame = CGRectMake(20, 100, 200, 50);
```

相关属性：

| 属性 | 坐标系 | 含义 |
| --- | --- | --- |
| `frame` | 父视图 | 外部位置和大小 |
| `bounds` | 自身 | 内部坐标和大小 |
| `center` | 父视图 | 中心点 |
| `transform` | 几何变换 | 旋转、缩放、平移 |

设置 `transform` 后，`frame` 是变换后的外接矩形，不适合作为后续精确布局依据。

### 4.2 Auto Layout

Auto Layout 使用约束描述视图关系：

```objc
self.button.translatesAutoresizingMaskIntoConstraints = NO;

[NSLayoutConstraint activateConstraints:@[
    [self.button.centerXAnchor constraintEqualToAnchor:self.view.centerXAnchor],
    [self.button.topAnchor constraintEqualToAnchor:self.view.safeAreaLayoutGuide.topAnchor constant:20]
]];
```

布局流程：

```text
updateConstraints
  ↓
layoutSubviews
  ↓
display
```

实践建议：

- 约束尽量创建一次，后续只改 `constant` 或 `priority`
- 不要在 `layoutSubviews` 中无条件创建约束
- 动态高度 Cell 要保证垂直方向约束闭环
- 复杂列表优先减少约束数量

### 4.3 `setNeedsLayout` 与 `layoutIfNeeded`

| API | 作用 | 是否立即执行 |
| --- | --- | --- |
| `setNeedsLayout` | 标记需要布局 | 否 |
| `layoutIfNeeded` | 如果需要布局则立即布局 | 是 |
| `setNeedsDisplay` | 标记需要重绘 | 否 |

约束动画：

```objc
self.widthConstraint.constant = 240;
[self.view setNeedsLayout];

[UIView animateWithDuration:0.25 animations:^{
    [self.view layoutIfNeeded];
}];
```

---

## 5. 绘制机制

### 5.1 `drawRect:`

当自定义绘制时，可以重写：

```objc
- (void)drawRect:(CGRect)rect {
    [[UIColor redColor] setFill];
    UIRectFill(rect);
}
```

注意：

- 不要手动直接调用 `drawRect:`
- 需要重绘时调用 `setNeedsDisplay`
- `drawRect:` 会触发 CPU 绘制，过重会卡顿
- 简单圆角、边框、颜色优先使用 layer 属性

### 5.2 backing store

需要绘制内容时，系统会生成 backing store，可以理解为 layer 的位图缓存。`drawRect:`、文本、图片等最终都会变成 GPU 可合成的纹理。

### 5.3 图片解码

图片显示前通常需要解码成位图。大图首次显示卡顿常见原因是主线程解码。

优化：

- 后台线程预解码
- 控制图片尺寸
- 使用缩略图
- 列表中按需加载和取消
- 复用缓存

---

## 6. 事件分发与响应者链

### 6.1 Hit-Testing

触摸事件从 `UIWindow` 开始向下寻找最合适的 View：

```text
UIWindow
  ↓ hitTest:withEvent:
RootView
  ↓ 倒序遍历 subviews
最深命中的 View
```

View 不参与命中测试的条件：

- `hidden == YES`
- `userInteractionEnabled == NO`
- `alpha <= 0.01`
- `pointInside:withEvent:` 返回 `NO`

扩大点击区域：

```objc
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    CGRect hitFrame = CGRectInset(self.bounds, -10, -10);
    return CGRectContainsPoint(hitFrame, point);
}
```

### 6.2 响应者链

命中 View 后，事件沿响应者链向上传递：

```text
Touched View
  ↓
Superview
  ↓
ViewController
  ↓
UIWindow
  ↓
UIApplication
```

可处理的事件包括：

- Touch
- Motion
- Remote Control
- Press

### 6.3 手势识别

`UIGestureRecognizer` 位于触摸事件和 View 响应之间，负责识别点击、滑动、拖拽、缩放等手势。

常见冲突处理：

```objc
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer
shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer {
    return YES;
}
```

---

## 7. 动画机制

### 7.1 隐式动画

直接修改 layer 属性可能触发隐式动画：

```objc
view.layer.opacity = 0.5;
```

UIKit 通常会在自己的动画 block 中管理事务：

```objc
[UIView animateWithDuration:0.25 animations:^{
    self.view.alpha = 0.5;
}];
```

### 7.2 显式动画

```objc
CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position.x"];
animation.fromValue = @0;
animation.toValue = @200;
animation.duration = 0.25;
[view.layer addAnimation:animation forKey:@"move"];
```

注意：显式动画只影响 Presentation Layer，不会自动修改 Model Layer。通常需要同时设置最终属性值。

### 7.3 CADisplayLink

`CADisplayLink` 与屏幕刷新同步，适合逐帧动画、游戏、渲染驱动：

```objc
self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(tick:)];
[self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
```

---

## 8. 离屏渲染与透明混合

### 8.1 离屏渲染

离屏渲染是 GPU 不能直接在当前 framebuffer 完成，需要先渲染到额外缓冲区，再合成回来。

常见触发：

- `mask`
- `layer.masksToBounds` + 圆角 + 复杂内容
- 动态阴影且没有 `shadowPath`
- `shouldRasterize`
- 某些视觉效果和滤镜

优化：

```objc
view.layer.shadowPath = [UIBezierPath bezierPathWithRect:view.bounds].CGPath;
```

### 8.2 透明混合

透明混合是 GPU 合成时需要读取下层像素再混合。

优化：

- 不透明视图设置 `opaque = YES`
- 减少半透明层级
- 避免大面积透明 PNG
- 列表 Cell 背景尽量不透明

### 8.3 光栅化

```objc
view.layer.shouldRasterize = YES;
view.layer.rasterizationScale = UIScreen.mainScreen.scale;
```

适合内容复杂但短时间不变的图层。不适合频繁变化的 Cell 或动画内容，否则会反复重建缓存。

---

## 9. 列表性能优化

### 9.1 Cell 复用

```objc
- (void)prepareForReuse {
    [super prepareForReuse];
    self.titleLabel.text = nil;
    self.avatarView.image = nil;
    self.representedIdentifier = nil;
}
```

异步加载图片要校验模型 ID：

```objc
NSString *identifier = model.identifier;
cell.representedIdentifier = identifier;

[loader load:model.url completion:^(UIImage *image) {
    if ([cell.representedIdentifier isEqualToString:identifier]) {
        cell.avatarView.image = image;
    }
}];
```

### 9.2 主线程预算

60 FPS 下每帧约 16.67ms，120 FPS 下每帧约 8.33ms。主线程中布局、解码、文本排版、业务计算都要争夺这段时间。

优化方向：

- 预计算 Cell 高度
- 后台解码图片
- 缓存排版结果
- 减少约束数量
- 按需加载
- 取消不可见 Cell 的异步任务

---

## 10. 线程安全

UIKit 不是线程安全的。所有 UI 更新必须在主线程：

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    self.label.text = @"Updated";
    [self.tableView reloadData];
});
```

可以在后台做：

- 数据解析
- 图片解码
- 文本尺寸计算
- 网络请求
- 数据库读取

但结果回到 UI 前必须切回主线程。

---

## 11. 调试工具

常用工具：

- Xcode View Debugger：查看 View/Layer 层级
- Instruments Core Animation：检查 FPS、离屏渲染、混合
- Time Profiler：定位主线程耗时
- Allocations：定位内存增长
- Debug Color Blended Layers：查看透明混合
- Debug Color Offscreen-Rendered：查看离屏渲染

---

## 12. 面试高频问题

1. **UIView 和 CALayer 的区别？**  
   UIView 负责事件、布局和层级管理；CALayer 负责渲染、动画和合成。

2. **为什么 UI 必须在主线程？**  
   UIKit 的事件、布局、渲染提交都围绕主线程 RunLoop 设计，跨线程修改会破坏状态一致性。

3. **`setNeedsLayout` 和 `layoutIfNeeded` 区别？**  
   前者标记异步布局，后者在需要布局时立即执行布局。

4. **离屏渲染为什么慢？**  
   它需要额外缓冲区和上下文切换，增加 GPU 工作量。

5. **圆角一定会离屏渲染吗？**  
   不一定。简单圆角通常可优化，但圆角裁剪叠加复杂内容、mask、阴影时容易触发。

6. **响应者链和 Hit-Test 的关系？**  
   Hit-Test 先向下找到事件接收 View，响应者链再从该 View 向上传递事件。

---

## 13. 总结

UIKit 渲染的核心是 UIView 管理结构和事件，CALayer 管理显示和动画，Core Animation 在 RunLoop 周期中提交事务，最终由 GPU 合成到屏幕。性能优化要围绕主线程预算、布局成本、图片解码、离屏渲染、透明混合和列表复用展开。

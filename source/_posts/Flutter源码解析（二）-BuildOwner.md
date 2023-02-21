---
title: Flutter源码解析（二）-BuildOwner
date: 2022-05-21 12:14:54
tags: 原理
categories: Flutter
---

BuildOwner 在 Element 状态管理上起到重要作用：

在 UI 更新过程中跟踪、管理需要 rebuild 的 Element (「dirty elements」);
在有「dirty elements」时，及时通知引擎，以便在下一帧安排上对「dirty elements」的 rebuild，从而去刷新 UI；
管理处于 "inactive" 状态的 Element。

<!-- more -->

这是我们遇到的第一个 Owner，后面还有 PipeOwner。

整棵「Element Tree」共享同一个 BuildOwner 实例 (全局的)，在 Element 挂载过程中由 parent 传递给 child element。

```dart
@mustCallSuper
void mount(Element parent, dynamic newSlot) {
  _parent = parent;
  _slot = newSlot;
  _depth = _parent != null ? _parent.depth + 1 : 1;
  _active = true;
  if (parent != null) // Only assign ownership if the parent is non-null
    _owner = parent.owner;
}
```

以上是 Element 基类的 mount 方法，第 8 行将 parent.owner 赋给了 child。

BuildOwner 实例由 WidgetsBinding 负责创建，并赋值给「Element Tree」的根节点 RenderObjectToWidgetElement，此后随着「Element Tree」的创建逐级传递给子节点。(具体流程后续文章会详细分析)
一般情况下并不需要我们手动实例化 BuildOwner，除非需要离屏沉浸 (此时需要构建 off-screen element tree)

BuildOwner 两个关键成员变量：

```dart
final _InactiveElements _inactiveElements = _InactiveElements();
final List<Element> _dirtyElements = <Element>[];
```

其命名已清晰表达了他们的用途：分别用于存储收集到的「Inactive Elements」、「Dirty Elements」。
Dirty Elements

那么 BuildOwner 是如何收集「Dirty Elements」的呢？
对于需要更新的 element，首先会调用 Element.markNeedsBuild 方法，如前文讲到的 State.setState 方法：

```dart
void setState(VoidCallback fn) {
  final dynamic result = fn() as dynamic;
  _element.markNeedsBuild();
}
```

如下，Element.markNeedsBuild 调用了 BuildOwner.scheduleBuildFor 方法：

```dart
void markNeedsBuild() {
  if (!_active)
    return;

  if (dirty)
    return;

  _dirty = true;
  owner.scheduleBuildFor(this);
}
```

BuildOwner.scheduleBuildFor 方法做了 2 件事：

调用 onBuildScheduled，该方法(其实是个 callback)会通知 Engine 在下一帧需要做更新操作；
将「Dirty Elements」加入到\_dirtyElements 中。

```dart
void scheduleBuildFor(Element element) {
  assert(element.owner == this);
  onBuildScheduled();

  _dirtyElements.add(element);
  element._inDirtyList = true;
}
```

此后，在新一帧绘制到来时，WidgetsBinding.drawFrame 会调用 BuildOwner.buildScope 方法：

```dart
void buildScope(Element context, [ VoidCallback callback ]) {
  if (callback == null && \_dirtyElements.isEmpty)
  return;

  try {
  if (callback != null) {
  callback();
}

    _dirtyElements.sort(Element._sort);
    int dirtyCount = _dirtyElements.length;
    int index = 0;
    while (index < dirtyCount) {
      _dirtyElements[index].rebuild();
      index += 1;
    }

} finally {
for (Element element in \_dirtyElements) {
element.\_inDirtyList = false;
}
\_dirtyElements.clear();
}
}
```

如有回调，先执行回调 (第 7 行)；
对「dirty elements」按在「Element Tree」上的深度排序 (即 parent 排在 child 前面) (第 10 行)；

为啥要这样排？确保 parent 先于 child 被 rebuild，以免 child 被重复 rebuild (因为 parent 在 rebuild 时会递归地 update child)。

对\_dirtyElements 中的元素依次调用 rebuild (第 14 行)；
清理\_dirtyElements (第 21 行)。

Inactive Elements

所谓「Inactive Element」，是指 element 从「Element Tree」上被移除到 dispose 或被重新插入「Element Tree」间的一个中间状态。
设计 inactive 状态的主要目的是实现『带有「global key」的 element』可以带着『状态』在树上任意移动。
BuildOwner 负责对「Inactive Element」进行管理，包括添加、删除以及对过期的「Inactive Element」执行 unmount 操作。
关于「Inactive Element」的更多信息将在介绍 Element 时一起介绍。

# 小结

BuildOwner 主要是用于收集那些需要 rebuild 的「Dirty Elements」以及处于 Inactive 状态的 Elements。
结束了！就是这么简单，下篇再见！

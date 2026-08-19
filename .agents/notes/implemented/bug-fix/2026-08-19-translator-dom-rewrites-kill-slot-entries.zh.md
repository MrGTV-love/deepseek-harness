# Agent Note：浏览器翻译器改写 React 托管 DOM 并击穿插槽条目

Status: implemented

[English](2026-08-19-translator-dom-rewrites-kill-slot-entries.md) | 中文

## 问题

一个开启了 DOM 翻译器的 Chrome 配置（内置翻译或沉浸式翻译等扩展）以两种方式破坏了会话输入区。输入区把草稿字形绘制在一个由原始文本节点组成的背景 `<div>` 中，旁边是透明 textarea 和隐藏的镜像层（[单一滚动端口](2026-07-31-composer-text-layers-share-one-scrollport.md)），因此改写文本节点的翻译器也在 React 之下改写背景层的子节点。

背景层尚在挂载时，React 持续更新已脱离文档的文本节点：按键被捕获、能提交，却无处绘制。提交时草稿清空，React 的删除提交试图移除一个早已被翻译器替换的子节点，提交抛出 `NotFoundError: Failed to execute 'removeChild' on 'Node'`。条目级错误边界捕获了异常并记录 `slot entry crashed in 'conversation.composer.bar'`；这个单值插槽条目永久让位，在输入条原位渲染无样式的空 `<div data-slot-error>`，直到刷新页面。

页面声明了 `<html lang="zh-CN">` 却没有任何翻译拒绝标记——当阅读者的界面语言与文档语言不一致时，翻译器正是针对这一状态动手的。

## 决策

`apps/web/index.html` 在 `<html>` 上声明 `translate="no"`，并携带 `<meta name="google" content="notranslate">`。应用自带 zh/en 语言切换，页面级翻译在这里没有正当职责；拒绝它一次性移除整类翻译器改写，而不是逐个插槽追堵。`InputBar` 的文本层栈（包裹背景层、textarea 与镜像层的 `.grow` 容器）额外携带自身的 `translate="no"`，这样即使在未采纳页面级拒绝的宿主文档中，草稿也保持原样。

插槽崩溃面现在可见。`packages/client/web/src/base.css` 为 `[data-slot-error]` 提供虚线错误描边、最小高度和由 `::before` 伪元素绘制的警告字形——伪元素内容不是文本节点，标记本身不会被翻译器改写。崩溃的条目无法再伪装成消失的输入区；失败会指名自己的插槽，直到刷新。

## 验证

组件测试套件覆盖了全部受守卫路径：输入区（`input-bar`，68 项）、含 IME 组合 Enter 的目标编辑器（`goalbar`），以及随本变更一并完成的弹出选择、队列停靠、工作区重命名与目录浏览器组合键守卫（六个套件共 239 项通过；`tsc -b tsconfig.client.json` 与仓库 oxlint 配置均无告警）。插槽崩溃面通过检查 `dsh web` 服务的构建样式表验证。

组装后的应用在复现过两个症状的浏览器配置（翻译器开启）下实测：多行草稿逐键可见，输入区在提交后存活。变更之前，同一配置几乎每次提交都会产生不可见草稿和消失的输入区。

## 考虑过的替代方案

**仅做插槽级 `translate="no"`，不做页面级拒绝。** 作为唯一手段被否决：内置翻译器与扩展各自认不同的提示，且输入区并非唯一的 React 托管文本。页面级拒绝是宽防线；输入区自身的属性是本地保证。

**为让位的单值插槽条目做恢复或重试。** 本变更中否决：让位机制的存在意义就是阻止崩溃条目循环。对意外崩溃，可见失败是最小且诚实的呈现；若将来出现真实的崩溃循环，恢复策略值得单独决策。

**拦截 React 的 DOM 变更。** 否决：框架拥有自己的提交路径，在框架层防御外来 DOM 改写，相对在源头拒绝改写而言不成比例。

## 后果

翻译器不再把文档级内容视为可翻译对象；希望换语言界面的阅读者使用内置语言切换。崩溃面样式作用于外壳中的每个插槽，未来任何条目崩溃都可见而非无声。输入区内的草稿文本永不成为翻译对象——它是用户原创输入。同一变更完成了 IME 组合键守卫扫尾（`isComposing`、旧式 `keyCode 229`，以及 Safari 先 compositionend 后 keydown 的时序，经延迟清除的 ref 实现），覆盖 `GoalBar`、`PopupSelectView`、`QueueDock`、`WorkspaceBrowser` 的两个重命名输入框和 `DirectoryBrowser` 的路径与文件夹输入框——这正是 `InputBar` 与 `QuestionComposer` 既有的守卫。

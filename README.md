

# 🚀 Gemini 模式自动切换器 (Tampermonkey)

**Gemini 模式自动切换器** 是一款专为提升 Google Gemini 使用体验而设计的油猴脚本。针对 Gemini 每次新对话都会默认跳转到 Fast 模式的问题，本插件可以实现**自动记忆并切换**至你心仪的模式，省去手动点击的烦扰。

### ✨ 功能亮点

* **自动切换**：进入页面即刻自动跳转至预设模式。
* **配置记忆**：通过悬浮窗一键设置，插件会记住你的选择。
* **直观操作**：页面内置悬浮控制窗，状态一目了然。

---

### 📺 视频教程

如果你是第一次使用油猴脚本，可以查看详细的视频演示：
👉 [点击查看教程视频](https://www.google.com/search?q=%E4%B8%80%E4%B8%AA%E7%A4%BA%E4%BE%8B%E9%93%BE%E6%8E%A5)

---

### 🛠️ 安装与使用指南

1. **获取源码**：滑动至本文末尾，复制提供的完整源代码。
2. **创建脚本**：
* 点击浏览器工具栏的 **Tampermonkey** 插件图标。
* 选择 “添加新脚本 (Create a new script)”。
* 清空编辑器内的原有内容，粘贴刚才复制的源码并**保存 (Ctrl+S)**。


3. **运行插件**：
* 打开或刷新 [Gemini 官网](https://gemini.google.com/)。
* 确保插件已处于“开启 (Enabled)”状态。


4. **模式切换**：
* 页面侧边会出现一个**悬浮控制窗**。
  ![悬浮窗示例](https://raw.githubusercontent.com/Bob8259/Gemini-/main/ScreenShot_2026-02-01_090131_012.png)
* 点击你需要的模式，插件将立即生效并在下次访问时自动应用。



---

### 📜 源代码

```
// ==UserScript==
// @name         Gemini 模式自动切换器
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  修复误点击自身UI的问题，完美自动切换
// @author       Azikaban/Bob
// @match        https://gemini.google.com/*
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        GM_addStyle
// ==/UserScript==

(function() {
    'use strict';

    // --- 1. 样式注入 ---
    GM_addStyle(`
        #gemini-selector-ui {
            position: fixed;
            top: 80px;
            right: 20px;
            background: rgba(255, 255, 255, 0.95);
            backdrop-filter: blur(5px);
            border: 1px solid #ccc;
            border-radius: 12px;
            padding: 12px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.15);
            z-index: 999999;
            font-family: sans-serif;
            display: flex;
            flex-direction: column;
            gap: 8px;
            min-width: 130px;
        }
        .ui-title { font-size: 13px; font-weight: bold; color: #555; margin-bottom: 5px; border-bottom: 1px solid #eee; padding-bottom: 5px; }
        .ui-label { display: flex; align-items: center; gap: 8px; font-size: 13px; cursor: pointer; color: #333; transition: color 0.2s; }
        .ui-label:hover { color: #1a73e8; }
        .ui-label input { margin: 0; cursor: pointer; }
    `);

    // --- 2. 状态管理 ---
    let targetMode = GM_getValue('gemini_target_mode', 'Thinking');
    let isSwitching = false;

    // --- 3. UI 构建 ---
    const injectUI = () => {
        if (document.getElementById('gemini-selector-ui')) return;

        const container = document.createElement('div');
        container.id = 'gemini-selector-ui';

        const title = document.createElement('div');
        title.className = 'ui-title';
        title.textContent = '自动模式设定';
        container.appendChild(title);

        const modes = [
            { id: 'Thinking', label: 'Thinking' },
            { id: 'Pro', label: 'Pro' },
            { id: 'OFF', label: '🔴关闭自动' }
        ];

        modes.forEach(mode => {
            const label = document.createElement('label');
            label.className = 'ui-label';

            const input = document.createElement('input');
            input.type = 'radio';
            input.name = 'gemini_mode';
            input.value = mode.id;
            input.checked = (targetMode === mode.id);

            input.addEventListener('change', (e) => {
                targetMode = e.target.value;
                GM_setValue('gemini_target_mode', targetMode);
                console.log('[Gemini] 切换目标已更改为:', targetMode);
            });

            const text = document.createElement('span');
            text.textContent = mode.label;

            label.appendChild(input);
            label.appendChild(text);
            container.appendChild(label);
        });

        (document.body || document.documentElement).appendChild(container);
    };

    // --- 4. 核心查找函数 (关键修复点) ---
    const findElementByText = (text, selector = 'span, div, button, li') => {
        const elements = document.querySelectorAll(selector);
        return Array.from(elements).find(el => {
            // 排除掉我们自己的 UI 面板 (#gemini-selector-ui) 里的元素
            if (el.closest('#gemini-selector-ui')) return false;

            const content = el.textContent.trim();
            // 宽松匹配：内容完全相等，或者包含且长度适中
            return content === text || (content.includes(text) && content.length < 35);
        });
    };

    // --- 5. 核心执行逻辑 ---
    const performCheckAndSwitch = () => {
        if (isSwitching || targetMode === 'OFF') return;

        // A. 定位当前模式按钮 (Angular 属性检测)
        const allSpans = document.querySelectorAll('span');
        let currentModeSpan = null;

        for (let span of allSpans) {
            const hasNgAttr = Array.from(span.attributes).some(attr => attr.name.startsWith('_ngcontent-ng'));
            const text = span.textContent.trim();

            if (hasNgAttr && text.length< 10 && (text.includes('Flash') || text.includes('Fast') || text.includes('Thinking') || text.includes('Pro'))) {
                currentModeSpan = span;
                break;
            }
        }

        if (!currentModeSpan) return;

        const currentText = currentModeSpan.textContent.trim();

        // B. 判断是否需要切换 (忽略大小写)
        if (!currentText.toLowerCase().includes(targetMode.toLowerCase())) {
            console.log(`[Gemini] 检测到模式不匹配: 当前[${currentText}] != 目标[${targetMode}]`);
            isSwitching = true;

            // 点击打开菜单
            const trigger = currentModeSpan.closest('button') || currentModeSpan.parentElement;
            trigger.click();

            // C. 查找并点击目标
            setTimeout(() => {
                const targetOption = findElementByText(targetMode);

                if (targetOption) {
                    console.log(`[Gemini] 找到目标选项 [${targetMode}]，点击！`);
                    targetOption.click();
                } else {
                    console.warn(`[Gemini] 未找到目标，可能是菜单未弹出或文字不匹配`);
                }

                // 冷却结束
                setTimeout(() => {
                    isSwitching = false;
                }, 2000);

            }, 300);
        }
    };

    // --- 6. 启动 ---
    setInterval(injectUI, 1000);
    setInterval(performCheckAndSwitch, 1000);

})();

```

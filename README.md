<!DOCTYPE html>
<html lang="zh-CN"><head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>透明背景 · 爱心悬浮祝福</title>
    <style>
        /* 彻底透明背景：body及所有父级透明，无任何底色 */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none; /* 避免文字选中，更像独立窗口 */
        }

        /* 完全透明背景 —— 关键点：body透明，html透明，且不设置任何背景色 */
        html, body {
            width: 100%;
            height: 100%;
            background: transparent !important;
            /* 确保没有任何背景层 */
            background-color: transparent;
            overflow: hidden;          /* 禁止滚动条，全屏沉浸但无背景 */
            position: relative;
        }

        body {
            /* 再次强制透明，并且允许穿透点击？不，卡片需要交互，但body背景无任何视觉元素 */
            background: transparent;
        }

        /* 模拟窗口卡片样式 —— 带柔和阴影、圆角，完全继承透明背景层 */
        .floating-card {
            position: fixed;
            width: 160px;
            height: 72px;
            border-radius: 20px;
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.12), 0 4px 8px rgba(0, 0, 0, 0.05);
            display: flex;
            align-items: center;
            justify-content: center;
            text-align: center;
            transition: all 0.2s ease;
            cursor: default;
            backdrop-filter: blur(0px);
            z-index: 1000;
            font-size: 14px;
            font-weight: 500;
            color: #1e2a3a;
            padding: 10px 8px;
            word-break: break-word;
            box-sizing: border-box;
            /* 初始透明且缩小，用于动画显现 */
            opacity: 0;
            transform: scale(0.85);
            pointer-events: auto;
            /* 优雅字体 */
            font-family: '微软雅黑', 'Segoe UI', 'PingFang SC', Roboto, 'Helvetica Neue', sans-serif;
            line-height: 1.35;
            letter-spacing: 0.3px;
        }

        /* 悬浮置顶高亮效果 (类似topmost) */
        .floating-card:hover {
            transform: scale(1.03);
            box-shadow: 0 16px 32px rgba(0, 0, 0, 0.2);
            z-index: 9999 !important;
            transition: all 0.2s cubic-bezier(0.2, 0.9, 0.4, 1.2);
        }

        /* 爱心阶段入场动画：渐现+弹入 */
        .card-appear {
            animation: gentlePop 0.22s ease forwards;
        }

        @keyframes gentlePop {
            0% {
                opacity: 0;
                transform: scale(0.7);
            }
            100% {
                opacity: 1;
                transform: scale(1);
            }
        }

        /* 散落窗口独特弹入动画 (更有活力) */
        .card-scatter {
            animation: scatterBounce 0.28s cubic-bezier(0.34, 1.2, 0.64, 1) forwards;
        }

        @keyframes scatterBounce {
            0% {
                opacity: 0;
                transform: scale(0.5) rotate(-4deg);
            }
            70% {
                opacity: 0.9;
                transform: scale(1.02) rotate(1deg);
            }
            100% {
                opacity: 1;
                transform: scale(1) rotate(0deg);
            }
        }

        /* 爱心隐藏淡出动画 (逐一消散) */
        .card-hide {
            animation: vanishOut 0.16s forwards !important;
        }

        @keyframes vanishOut {
            to {
                opacity: 0;
                transform: scale(0.6);
                visibility: hidden;
            }
        }
        
        /* 确保body任何角落无背景图、无颜色 */
        body::before, body::after {
            display: none;
        }
    </style>
</head>
<body>
<script>
    // ========== 预设显示内容与背景色 (保留原版暖色风格) ==========
    const textOptions = [
        "早安，新的一天开始啦",
        "保持热爱", "奔赴山海",
        "平安喜乐", "万事胜意",
        "努力上进", "闪闪发光",
        "生活明朗", "万物可爱",
        "好运常在", "笑口常开"
    ];

    const bgOptions = [
        "#f8e9e9", "#e9f8f8", "#f8f5e9",
        "#e9f2f8", "#f2e9f8", "#f8e9f2"
    ];

    // ---------- 辅助函数 ----------
    function getRandomItem(arr) {
        return arr[Math.floor(Math.random() * arr.length)];
    }

    function getRandomBg() {
        return getRandomItem(bgOptions);
    }

    function getRandomText() {
        return getRandomItem(textOptions);
    }

    // ---------- 心形参数方程 (标准爱心) ----------
    // 返回爱心轮廓点集 (基于中心坐标与缩放)
    function getHeartPoints(centerX, centerY, size, numPoints) {
        const points = [];
        for (let i = 0; i < numPoints; i++) {
            const t = 2 * Math.PI * i / numPoints;
            // 标准心形公式: x = 16 * sin(t)^3, y = 13cos(t) - 5cos(2t) - 2cos(3t) - cos(4t)
            const xStd = 10 * Math.pow(Math.sin(t), 3);
            const yStd = 5/8*(13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t));
            const x = centerX + xStd * size - 72;
            const y = centerY - yStd * size + 36;   // 屏幕Y轴向下，减去使爱心正向
            points.push({ x: Math.round(x), y: Math.round(y) });
        }
        return points;
    }

    // ---------- 核心管理器 (透明背景 + 完整流程) ----------
    class TransparentHeartManager {
        constructor() {
            this.heartWindows = [];      // 爱心阶段DOM元素
            this.scatteredWindows = [];  // 散落阶段DOM元素
            this.screenWidth = window.innerWidth;
            this.screenHeight = window.innerHeight;
            this.scatterAutoCloseTimer = null;
        }

        // 更新屏幕尺寸
        updateScreenSize() {
            this.screenWidth = window.innerWidth;
            this.screenHeight = window.innerHeight;
        }

        // 创建单个卡片 (透明背景下的柔和卡片)
        createCard(text, bgColor, leftPos, topPos, phase = 'heart') {
            const card = document.createElement('div');
            card.className = 'floating-card';
            card.style.backgroundColor = bgColor;
            // 确保卡片背景完全不透明以保证文字可读，但网页背景透明 (满足需求)
            card.style.left = `${leftPos}px`;
            card.style.top = `${topPos}px`;
            // 添加文字，支持换行
            card.innerHTML = `<span style="display:block; font-weight:500;">${escapeHtml(text)}</span>`;
            
            // 悬浮置顶效果: 鼠标进入提升层级，类似 attributes('-topmost', True) 的模拟
            card.addEventListener('mouseenter', () => {
                card.style.zIndex = '10000';
                card.style.transition = 'transform 0.15s, box-shadow 0.2s';
            });
            card.addEventListener('mouseleave', () => {
                card.style.zIndex = '1000';
            });
            return card;
        }

        // 睡眠函数 (延迟)
        sleep(ms) {
            return new Promise(resolve => setTimeout(resolve, ms));
        }

        // 生成爱心轮廓窗口 (顺时针/逐一创建)
        async createHeartWindows() {
            this.updateScreenSize();
            const centerX = this.screenWidth / 2;
            const centerY = this.screenHeight / 2;
            const heartSize = 18;        // 爱心缩放系数
            const numWindows = 64;        // 爱心点数 (与原来一致)
            
            const positions = getHeartPoints(centerX, centerY, heartSize, numWindows);
            
            // 逐一创建窗口，延迟间隔 0.05s (50ms左右)
            for (let i = 0; i < positions.length; i++) {
                const pos = positions[i];
                const text = getRandomText();
                const bgColor = getRandomBg();
                
                const card = this.createCard(text, bgColor, pos.x, pos.y, 'heart');
                document.body.appendChild(card);
                this.heartWindows.push(card);
                // 添加入场动画
                card.classList.add('card-appear');
                
                // 等待一小段时间，模拟窗口顺序弹出
                await this.sleep(100);
            }
            
            // 爱心全部出现后等待3秒 (与原代码一致)
            await this.sleep(3000);
            await this.hideHeartWindowsSequentially();
        }
        
        // 逐一隐藏爱心窗口 (消散动画，类似 withdraw)
        async hideHeartWindowsSequentially() {
            // 按照创建顺序逐个隐藏
            const windows = [...this.heartWindows];
            for (let i = 0; i < windows.length; i++) {
                const win = windows[i];
                if (win && win.parentNode) {
                    win.classList.add('card-hide');
                    await this.sleep(20);  // 隐藏间隔 20ms 与原版0.02s一致
                    if (win.parentNode) win.remove();
                }
            }
            this.heartWindows = [];
            // 稍作停顿再生成散落窗口 (符合原 python after 200)
            await this.sleep(200);
            this.createScatteredWindows();
        }
        
        // 创建散落窗口 (随机位置，总计64个)
        async createScatteredWindows() {
            this.updateScreenSize();
            const numScattered = 64;
            const newWindows = [];
            
            // 避免边缘溢出，确保窗口完全可见
            const maxLeft = Math.max(0, this.screenWidth - 200);
            const maxTop = Math.max(0, this.screenHeight - 90);
            
            for (let i = 0; i < numScattered; i++) {
                const text = getRandomText();
                const bg = getRandomBg();
                const randX = Math.floor(Math.random() * maxLeft);
                const randY = Math.floor(Math.random() * maxTop);
                
                const card = this.createCard(text, bg, randX, randY, 'scatter');
                document.body.appendChild(card);
                card.classList.add('card-scatter');
                newWindows.push(card);
                await this.sleep(45);   // 与原版间隔一致
            }
            
            this.scatteredWindows = newWindows;
            
            // 等待10秒后自动关闭所有散落窗口并结束程序 (与原逻辑一致)
            this.scatterAutoCloseTimer = setTimeout(() => {
                this.autoCloseAll();
            }, 10000);
        }
        
        // 自动关闭所有窗口 + 清理 (完全退出效果)
        autoCloseAll() {
            // 优雅移除散落窗口
            for (let win of this.scatteredWindows) {
                if (win && win.parentNode) {
                    win.style.transition = 'opacity 0.2s, transform 0.15s';
                    win.style.opacity = '0';
                    win.style.transform = 'scale(0.7)';
                    setTimeout(() => {
                        if (win.parentNode) win.remove();
                    }, 150);
                }
            }
            // 清除爱心残留 (确保万无一失)
            for (let win of this.heartWindows) {
                if (win && win.parentNode) win.remove();
            }
            // 彻底清空后不留任何额外元素，但保留完全透明背景
            this.scatteredWindows = [];
            this.heartWindows = [];
        }
        
        // 完全重置并启动主流程
        async start() {
            this.clearAllWindows();
            await this.createHeartWindows();
        }
        
        clearAllWindows() {
            if (this.scatterAutoCloseTimer) clearTimeout(this.scatterAutoCloseTimer);
            const allCards = document.querySelectorAll('.floating-card');
            allCards.forEach(card => card.remove());
            this.heartWindows = [];
            this.scatteredWindows = [];
        }
    }
    
    // 简单XSS防护 (转义html)
    function escapeHtml(str) {
        return str.replace(/[&<>]/g, function(m) {
            if (m === '&') return '&amp;';
            if (m === '<') return '&lt;';
            if (m === '>') return '&gt;';
            return m;
        }).replace(/[\uD800-\uDBFF][\uDC00-\uDFFF]/g, function(c) {
            return c;
        });
    }
    
    // 实例化并启动
    let manager = null;
    
    function init() {
        manager = new TransparentHeartManager();
        // 稍微延迟确保渲染稳定
        setTimeout(() => {
            manager.start().catch(e => console.warn(e));
        }, 100);
    }
    
    // 页面完全透明背景启动
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', init);
    } else {
        init();
    }
    
    // 监听窗口大小改变：当在爱心阶段还未完成时如果resize，可能导致爱心位置偏移，但原python tkinter固定尺寸不会重绘，我们为了体验
    // 仅在未生成散落且爱心阶段刚开始时不做破坏性更新，因为用户极少大幅缩放；若需要完美，可简单忽略，保持原位置也无伤大雅。
    // 但为了避免散落窗口超出边界，在resize时已经动态获取新的宽高仅用于后续散落窗口生成（已做max限制）爱心位置在生成时已锁定，完全合理。
    // 额外确保透明背景始终无残留背景色
    function enforceTransparentBackground() {
        document.documentElement.style.backgroundColor = 'transparent';
        document.body.style.backgroundColor = 'transparent';
    }
    enforceTransparentBackground();
    // 监听动态确保透明（例如某些浏览器插件干扰）
    setInterval(enforceTransparentBackground, 500);
</script>
</body>
</html>

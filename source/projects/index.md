---
title: 个人作品
date: 2026-10-13 12:00:00
comments: false
---

<style>
.projects-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 20px;
  margin: 24px 0;
}
.project-card {
  display: flex;
  flex-direction: column;
  border: 1px solid var(--color-border);
  border-radius: 4px;
  overflow: hidden;
  background-color: var(--color-bg);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}
.project-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.12);
}

/* 媒体舞台：所有作品共用同一比例，媒体按 contain 居中 */
.projects-grid .project-stage,
.projects-grid .carousel-viewport {
  position: relative;
  aspect-ratio: 4 / 3;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  background-color: var(--color-hover);
}
.projects-grid .carousel-viewport {
  display: block;
}
.projects-grid .project-stage video {
  display: block;
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  margin: 0;
  object-fit: contain;
}

/* 说明条：两种媒体共用同一样式 */
.projects-grid .project-caption,
.projects-grid .carousel-caption {
  margin: 0;
  padding: 9px 14px;
  border-top: 1px solid var(--color-border);
  font-size: 12.5px;
  line-height: 1.6;
  color: var(--color-secondary);
}

.projects-grid .project-body {
  flex: 1;
  padding: 12px 14px 14px;
}
.projects-grid .project-title {
  margin: 0 0 6px;
  font-size: 16px;
  font-weight: 600;
  line-height: 1.4;
}
.projects-grid .project-desc {
  margin: 0;
  font-size: 13px;
  line-height: 1.6;
  color: var(--color-secondary);
}
.project-link {
  display: inline-block;
  margin-top: 10px;
  font-size: 13px;
}

/* 跑马灯 */
.projects-grid .carousel-track {
  display: flex;
  height: 100%;
  transition: transform 0.4s ease;
}
.projects-grid .carousel-slide {
  flex: 0 0 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  padding-bottom: 36px;
  box-sizing: border-box;
}
.projects-grid .carousel-slide img {
  display: block;
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  margin: 0;
  object-fit: contain;
  /* 截图自带浅色底，加边框让留白读作刻意的衬底而非尺寸失误 */
  border: 1px solid var(--color-border);
  border-radius: 3px;
}
.projects-grid .carousel-btn {
  position: absolute;
  top: calc(50% - 18px);
  transform: translateY(-50%);
  z-index: 2;
  width: 36px;
  height: 36px;
  padding: 0;
  border: none;
  border-radius: 50%;
  background-color: rgba(0, 0, 0, 0.45);
  color: #fff;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  transition: background-color 0.25s ease;
}
.projects-grid .carousel-btn:hover {
  background-color: rgba(0, 0, 0, 0.7);
}
.projects-grid .carousel-prev { left: 12px; }
.projects-grid .carousel-next { right: 12px; }
.projects-grid .carousel-dots {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 12px;
  z-index: 2;
  display: flex;
  justify-content: center;
}
.projects-grid .carousel-dot {
  width: 44px;
  height: 44px;
  padding: 18px;
  border: none;
  border-radius: 50%;
  background-color: var(--color-border);
  background-clip: content-box;
  cursor: pointer;
  transition: background-color 0.25s ease, transform 0.25s ease;
}
.projects-grid .carousel-dot.active {
  background-color: var(--color-primary);
  transform: scale(1.35);
}

/* 键盘焦点可见 */
.projects-grid :focus-visible {
  outline: 2px solid var(--color-link);
  outline-offset: 2px;
}

@media (prefers-color-scheme: dark) {
  .projects-grid .project-caption,
  .projects-grid .carousel-caption {
    color: #a8a8a8;
  }
}

@media screen and (max-width: 600px) {
  /* 竖屏视频在窄屏改用自身比例，避免被舞台压成小图 */
  .projects-grid .project-stage--portrait {
    aspect-ratio: auto;
  }
  .projects-grid .project-stage--portrait video {
    width: 100%;
    max-height: none;
  }
}

@media (prefers-reduced-motion: reduce) {
  .projects-grid .project-card,
  .projects-grid .carousel-track,
  .projects-grid .carousel-btn,
  .projects-grid .carousel-dot {
    transition: none;
  }
}
</style>

<div class="projects-grid">

  <article class="project-card">
    <div class="carousel" data-carousel>
      <div class="carousel-viewport" role="region" aria-roledescription="轮播" aria-label="单词乐乐园界面截图">
        <div class="carousel-track">
          <div class="carousel-slide" data-caption="探索页 · 追踪听捕、火眼金睛、拼装怪兽三种玩法，左侧是宠物养成与能量豆进度">
            <img src="/projects/index/word-fun-1-explore.jpg" alt="单词乐乐园 探索页">
          </div>
          <div class="carousel-slide" data-caption="词库 · 支持从 PEP 等教材按单元批量勾选导入，也可以自己新建词库">
            <img src="/projects/index/word-fun-2-textbook.jpg" alt="单词乐乐园 教材导入">
          </div>
          <div class="carousel-slide" data-caption="成就 · 每个单词分听力、翻译、拼写三项考察，全部集齐才算点亮">
            <img src="/projects/index/word-fun-3-achievements.jpg" alt="单词乐乐园 成就页">
          </div>
          <div class="carousel-slide" data-caption="Debug 页 · 内部管理入口，可预览全部宠物形态、清空数据、开关尚未完成的宠物">
            <img src="/projects/index/word-fun-4-debug.jpg" alt="单词乐乐园 Debug 管理页">
          </div>
        </div>
        <button class="carousel-btn carousel-prev" type="button" aria-label="上一张">‹</button>
        <button class="carousel-btn carousel-next" type="button" aria-label="下一张">›</button>
        <div class="carousel-dots"></div>
      </div>
      <p class="carousel-caption" aria-live="polite"></p>
    </div>
    <div class="project-body">
      <h2 class="project-title">单词乐乐园</h2>
      <p class="project-desc">给小学孩子背单词的纯前端网站，内置两个怪兽等级，包含听力、翻译、拼写三个能力的游戏。</p>
      <a class="project-link" href="https://word-fun.nnsay.cn/" target="_blank" rel="noopener">访问网站 →</a>
    </div>
  </article>

  <article class="project-card">
    <div class="project-stage project-stage--portrait">
      <video src="/projects/index/appilot-demo.mp4"
             poster="/projects/index/appilot-poster.jpg"
             aria-label="appilot 演示录屏：在 Boss 直聘 App 上自动采集职位并完成首轮沟通"
             controls muted loop playsinline preload="metadata"></video>
    </div>
    <p class="project-caption">演示 · 采集职位 → 简历匹配评估 → 生成打招呼消息，已沟通职位自动去重</p>
    <div class="project-body">
      <h2 class="project-title">appilot · Boss 直聘自动沟通 Agent</h2>
      <p class="project-desc">在 Boss 直聘 App 上自动采集职位、按简历匹配度评估并完成首轮沟通的 AI Agent。通过 ADB + UI Tree 驱动真实设备，绕开网页版对 CDP 自动化的驱动层检测；流程由 LangGraph 状态机编排，LLM 以结构化输出判定匹配并生成打招呼消息。</p>
      <a class="project-link" href="https://github.com/nnsay/appilot" target="_blank" rel="noopener">查看源码 →</a>
    </div>
  </article>

</div>

<script>
(function () {
  document.querySelectorAll('[data-carousel]').forEach(function (root) {
    var track = root.querySelector('.carousel-track');
    var slides = Array.prototype.slice.call(root.querySelectorAll('.carousel-slide'));
    var dotsBox = root.querySelector('.carousel-dots');
    var caption = root.querySelector('.carousel-caption');
    var index = 0;

    var dots = slides.map(function (slide, i) {
      var dot = document.createElement('button');
      dot.type = 'button';
      dot.className = 'carousel-dot';
      dot.setAttribute('aria-label', '第 ' + (i + 1) + ' 张');
      dot.addEventListener('click', function () { go(i); });
      dotsBox.appendChild(dot);
      return dot;
    });

    function go(next) {
      index = (next + slides.length) % slides.length;
      track.style.transform = 'translateX(-' + index * 100 + '%)';
      dots.forEach(function (dot, i) {
        dot.classList.toggle('active', i === index);
        dot.setAttribute('aria-current', i === index ? 'true' : 'false');
      });
      // 离屏 slide 对屏幕阅读器隐藏，避免读出四张无标签截图
      slides.forEach(function (slide, i) {
        if (i === index) { slide.removeAttribute('aria-hidden'); }
        else { slide.setAttribute('aria-hidden', 'true'); }
      });
      caption.textContent = slides[index].getAttribute('data-caption') || '';
    }

    root.querySelector('.carousel-prev').addEventListener('click', function () { go(index - 1); });
    root.querySelector('.carousel-next').addEventListener('click', function () { go(index + 1); });

    // 方向键：焦点在轮播内时可用
    root.addEventListener('keydown', function (e) {
      if (e.key === 'ArrowLeft') { go(index - 1); }
      else if (e.key === 'ArrowRight') { go(index + 1); }
      else { return; }
      e.preventDefault();
    });

    var startX = null;
    root.addEventListener('touchstart', function (e) {
      startX = e.touches[0].clientX;
    }, { passive: true });
    root.addEventListener('touchend', function (e) {
      if (startX === null) return;
      var dx = e.changedTouches[0].clientX - startX;
      if (Math.abs(dx) > 40) go(dx < 0 ? index + 1 : index - 1);
      startX = null;
    });

    go(0);
  });
})();
</script>

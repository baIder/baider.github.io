---
title: JS&CSS demos
date: 2023-05-12 11:18:38
---

# 收录一些仅用JS和CSS实现的效果

## 树叶下落的效果

链接地址：https://codepen.io/bald3r/pen/vYVrKrv?editors=1100

```html
<body>
  <section>
    <h3>下落的落叶背景效果</h3>
    <div class="set">
      <div><img src="https://img.bald3r.wang/img/202305121403298.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403531.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403719.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403949.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403298.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403531.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403719.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403949.png" alt=""></div>
    </div>
    <div class="set set2">
      <div><img src="https://img.bald3r.wang/img/202305121403298.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403531.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403719.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403949.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403298.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403531.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403719.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403949.png" alt=""></div>
    </div>
    <div class="set set3">
      <div><img src="https://img.bald3r.wang/img/202305121403298.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403531.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403719.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403949.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403298.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403531.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403719.png" alt=""></div>
      <div><img src="https://img.bald3r.wang/img/202305121403949.png" alt=""></div>
    </div>
  </section>
</body>
```

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: "Popoins", sans-serif;
}

section {
  position: relative;
  width: 100%;
  height: 100vh;
  background: radial-gradient(#333, #000);
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

section h3 {
  color: #fff;
  font-size: 4em;
  text-transform: uppercase;
}

section .set {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  pointer-events: none;
}

.set2 {
  transform: scale(2) rotateY(180deg);
  filter: blur(2px);
}

.set3 {
  transform: scale(0.8) rotateY(180deg);
  filter: blur(4px);
}

section .set div {
  position: absolute;
  display: block;
}

section .set div:nth-child(1) {
  left: 20%;
  animation: animate 15s linear infinite;
  animation-delay: -7s;
}

section .set div:nth-child(2) {
  left: 50%;
  animation: animate 20s linear infinite;
  animation-delay: -5s;
}

section .set div:nth-child(3) {
  left: 70%;
  animation: animate 20s linear infinite;
  animation-delay: 0s;
}

section .set div:nth-child(4) {
  left: 0%;
  animation: animate 15s linear infinite;
  animation-delay: -5s;
}

section .set div:nth-child(5) {
  left: 85%;
  animation: animate 18s linear infinite;
  animation-delay: -10s;
}

section .set div:nth-child(6) {
  left: 0;
  animation: animate 12s linear infinite;
  animation-delay: 0s;
}

section .set div:nth-child(7) {
  left: 15%;
  animation: animate 14s linear infinite;
  animation-delay: 0s;
}

section .set div:nth-child(8) {
  left: 90%;
  animation: animate 18s linear infinite;
  animation-delay: -1s;
}


@keyframes animate {
  0% {
    opacity: 0;
    top: -10%;
    transform: translateX(20px) rotate(0deg);
  }
  10% {
    opacity: 1;
  }
  20% {
    transform: translateX(-20px) rotate(45deg);
  }
  40% {
    transform: translateX(-20px) rotate(90deg);
  }
  60% {
    transform: translateX(20px) rotate(180deg);
  }
  80% {
    transform: translateX(-20px) rotate(180deg);
  }
  100% {
    top: 110%;
    transform: translateX(-20px) rotate(226deg);
  }

}
```
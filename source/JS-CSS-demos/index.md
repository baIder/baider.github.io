---
title: JS&CSS demos
date: 2023-05-12 11:18:38
---

# 收录一些仅用JS和CSS实现的效果

从各大网站转载我喜欢的前端效果


---

## 树叶下落的效果

链接地址：https://codepen.io/bald3r/pen/vYVrKrv?editors=1100

+++ **显示代码**

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

+++

---

## 带动画的菜单

链接地址：https://codepen.io/bald3r/pen/VwEEzzd?editors=0010

+++ **显示代码**

```html
<div class="navigation">
  <ul id="nav">
    <li class="list active">
      <a href="#">
        <span class="icon"><ion-icon name="home-outline"></ion-icon></span>
        <span class="text">Home</span>
      </a>
    </li>
    <li class="list">
      <a href="#">
        <span class="icon"
          ><ion-icon name="person-outline"></ion-icon
        ></span>
        <span class="text">Profile</span>
      </a>
    </li>
    <li class="list">
      <a href="#">
        <span class="icon"
          ><ion-icon name="chatbubble-outline"></ion-icon
        ></span>
        <span class="text">Messages</span>
      </a>
    </li>
    <li class="list">
      <a href="#">
        <span class="icon"
          ><ion-icon name="camera-outline"></ion-icon
        ></span>
        <span class="text">Photos</span>
      </a>
    </li>
    <li class="list">
      <a href="#">
        <span class="icon"
          ><ion-icon name="settings-outline"></ion-icon
        ></span>
        <span class="text">Settings</span>
      </a>
    </li>
    <div class="indicator"></div>
  </ul>
</div>

<script type="module" src="https://unpkg.com/ionicons@5.5.2/dist/ionicons/ionicons.esm.js"></script>
<script nomodule src="https://unpkg.com/ionicons@5.5.2/dist/ionicons/ionicons.js"></script>

```

```css

@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@100;200;300;400;500;600;700;800;900&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Poppins', sans-serif;
}

:root {
  --clr: #393939;
}
body {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background: var(--clr);
}

.navigation {
  width: 400px;
  height: 70px;
  background: #e4dccf;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 10px;
}

.navigation ul {
  display: flex;
  width: 350px;
}

.navigation ul li {
  position: relative;
  list-style: none;
  width: 70px;
  height: 70px;
  z-index: 1;
}

.navigation ul li a {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  width: 100%;
  text-align: center;
  font-weight: 500;
}

.navigation ul li a .icon {
  position: relative;
  display: block;
  line-height: 75px;
  font-size: 1.5em;
  text-align: center;
  transition: 250ms;
  color: var(--clr);
}

.navigation ul li.active a .icon {
  transform: translateY(-32px);
}

.navigation ul li a .text {
  position: absolute;
  color: var(--clr);
  font-weight: 400;
  font-size: 0.75em;
  letter-spacing: 0.05em;
  transition: 250ms;
  opacity: 0;
  transform: translateY(20px);
}

.navigation ul li.active a .text {
  opacity: 1;
  transform: translateY(10px);
}

.indicator {
  position: absolute;
  top: -50%;
  width: 70px;
  height: 70px;
  background: #d2ebcd;
  border-radius: 50%;
  border: 6px solid var(--clr);
  transition: 250ms;
}

.indicator::before {
  content: '';
  position: absolute;
  top: 50%;
  left: -22px;
  width: 20px;
  height: 20px;
  background: transparent;
  border-top-right-radius: 20px;
  box-shadow: 1px -10px 0 0 var(--clr);
}

.indicator::after {
  content: '';
  position: absolute;
  top: 50%;
  right: -22px;
  width: 20px;
  height: 20px;
  background: transparent;
  border-top-left-radius: 20px;
  box-shadow: -1px -10px 0 0 var(--clr);
}

.navigation ul li:nth-child(1).active ~ .indicator {
  transform: translateX(calc(70px * 0));
}

.navigation ul li:nth-child(2).active ~ .indicator {
  transform: translateX(calc(70px * 1));
}
.navigation ul li:nth-child(3).active ~ .indicator {
  transform: translateX(calc(70px * 2));
}
.navigation ul li:nth-child(4).active ~ .indicator {
  transform: translateX(calc(70px * 3));
}
.navigation ul li:nth-child(5).active ~ .indicator {
  transform: translateX(calc(70px * 4));
}


```

```js
const list = document.querySelectorAll(".list");
const ul = document.getElementById("nav");
ul.addEventListener("click", (e) => activeLink(e));
function activeLink(e) {
  list.forEach((item) => {
    item.classList.remove("active");
  });
  let target = e.target;
  while (target.tagName.toLowerCase() !== "ul") {
    if (target.tagName.toLowerCase() === "li") {
      break;
    } else {
      target = target.parentNode;
    }
  }
  target.classList.add("active");
}

```

+++

---
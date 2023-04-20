---
title: 发财图床 React懒加载和加载中
date: 2022-08-13 15:29:14
tags: 
  - React
---

# \[发财图床] React懒加载和加载中

## 前言

最近在做一个`React`+`React Router`+`TypeScript`+`Mobx`+`LeanCloud`的一个图床项目，

为了提高性能，在页面的展示中使用了懒加载，并做了一个加载页面，本文将记录一下实现过程

## 加载页面的制作

因为是在项目搭建骨架的时候制作的，所以有些草率，主要是那个意思

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/GIF%202022-8-13%2016-01-05_aBwFgF1DUY.gif)

## React懒加载

```react
import React, {lazy, Suspense} from 'react';
import './App.css';
import Header from './components/Header';
import Footer from './components/Footer';
import {Routes, Route} from 'react-router-dom';
import Loading from './components/Loading';
import Login from './pages/Login';
import Register from './pages/Register';

const Home = lazy(() => import('./pages/Home'));
const History = lazy(() => import('./pages/History'));
const About = lazy(() => import('./pages/About'));

function App() {
  return (
    <>
      <Header/>
      <main>
        <Suspense fallback={<Loading/>}>
          <Routes>
            <Route path="/" element={<Home/>}/>
            <Route path="/history" element={<History/>}/>
            <Route path="/about" element={<About/>}/>
            <Route path="/login" element={<Login/>}/>
            <Route path="/register" element={<Register/>}/>
          </Routes>
        </Suspense>
      </main>
      <Footer/>
    </>
  );
}

export default App;
```

主要分为下面几个步骤：

1.  需要懒加载的组件通过`lazy(()=>import())`动态加载
2.  路由通过`<Suspense></Suspense>`标签包裹
3.  `<Suspense fallback={}></Suspense>`中`fallback={}`就是在加载过程中调用的内容，可以是函数或者React组件，本项目中使用的是组件

## 后记

如果你有更好的实现方法，欢迎留言告诉我\~

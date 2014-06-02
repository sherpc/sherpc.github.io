---
layout: post
title: "Quiescent &amp; Core.Async &amp; WebSockets"
date: 2014-06-02 23:03:41 +0400
comments: true
categories: [clojurescript, react.js]
---

## Цель

Создать клиентское приложение с помощью clojure script, core.async & web sockets, которое будет слушать данные с сервера и показывать их в виде графика в реальном времени.

## Часть I -- Quiescent

Quiescent предназначен для рендеринга dom-дерева через React.js. Общая идея: получаем на вход некое состояние (обычно clojurescript map), и выдаем его в виде dom-дерева. В отличие от Om, quiescent не предназначен для обработки dom-событий.

Для примеров я буду использовать LightTable. Скачать можно здесь: http://www.lighttable.com/

Также, нам нужен установленый leiningen. Берем отсюда: https://github.com/technomancy/leiningen#installation

Создаем проект для работы с ClojureScript, пользуясь подсказками из этой статьи: http://yobriefca.se/blog/2014/05/30/basic-clojurescript-setup/

``` bash
lein new mies <project name>
cd <project name>

```

``` clojure 
(go (loop []
      (println (<! in-channel-one))
      (recur)))
```




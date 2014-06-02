---
layout: post
title: "Quiescent &amp; Core.Async &amp; WebSockets"
date: 2014-06-02 23:03:41 +0400
comments: true
categories: [clojurescript, react.js]
---

## Цель

Создать клиентское приложение с помощью clojure script, core.async & web sockets, которое будет слушать данные с сервера и показывать их в виде графика в реальном времени.

## Часть 0 -- Подготовка

Quiescent предназначен для рендеринга dom-дерева через React.js. Общая идея: получаем на вход некое состояние (обычно clojurescript map), и выдаем его в виде dom-дерева. В отличие от Om, quiescent не предназначен для обработки dom-событий.

Для примеров я буду использовать LightTable. Скачать можно здесь: http://www.lighttable.com/

Также, нам нужен установленый leiningen. Берем отсюда: https://github.com/technomancy/leiningen#installation

### Настройка проекта с ClojureScript

Создаем проект для работы с ClojureScript, пользуясь подсказками из этой статьи: http://yobriefca.se/blog/2014/05/30/basic-clojurescript-setup/ (я назвал проект random-charts):

``` bash
sherpc@javari:~/Projects$ lein new mies random-charts
sherpc@javari:~/Projects$ cd random-charts
sherpc@javari:~/Projects/random-charts$ cat > Procfile
web: lein simpleton 8080
cljs: lein cljsbuild auto
sherpc@javari:~/Projects/random-charts$ 
```
Правим project.clj:

``` clojure project.clj 
(defproject random-charts "0.1.0-SNAPSHOT"
  :description "FIXME: write this!"
  :url "http://example.com/FIXME"

  :dependencies [[org.clojure/clojure "1.5.1"]
                 [org.clojure/clojurescript "0.0-2173"]]

  :plugins [[lein-cljsbuild "1.0.2"]
            [lein-simpleton "1.3.0"]
            [lein-cooper "0.0.1"]]

  :source-paths ["src"]

  :cljsbuild { 
    :builds [{:id "random-charts"
              :source-paths ["src"]
              :compiler {
                :output-to "random_charts.js"
                :output-dir "out"
                :optimizations :none
                :source-map true}}]})

```

Запускаем cooper и проверяем консоль по адресу http://localhost:8080

``` bash
sherpc@javari:~/Projects/random-charts$ lein cooper
```
{% img http://snapr.pw/i/cdec198d80.png %}

Ура, у нас настроено рабочее окружение для clojure script. 

### Подключаем REPL

Теперь подключим LightTable к нашей вкладке в браузере. Для этого открываем Connect bar (crl+space show connect...), выбираем Add -> Browser (External) и копируем script-тег, который покажет LightTable. Вставляем этот тег в index.html:
{% include_code index.html lang:html index.html %}
Обновляем страницу в браузере (F5), и в LightTable внизу в строке статуса видим Connected to localhost:8080. Чтобы проверить, наберем в файле core.cljs следующую строчку, выделим её и нажмем ctrl+enter (eval selected):
``` clojure
(js/alert "Hello!")
```
LightTable первый раз долго подключается к проекту, и в итоге в браузере мы должны увидеть заветный alert с текстом hello!. Попробуем println:
``` clojure
(println (map inc [1 2 3]))
```
В консоли видим (2 3 4) -- ура, все работает! Переходим к самому интересному.

### Немного стилей
Подключаем bootstrap (http://getbootstrap.com/) в index.html:
{% include_code index.html lang:html index-bootstrap.html %}
И добавляем файл css/style.css:
``` css
body {
  padding-top: 60px;
}
```

## Часть I -- Quiescent

Не забываем подключить сам React.js в index.html.

### Простой компонент

Для начала сделаем компонент, который просто реализует alert из bootstrap (http://getbootstrap.com/components/#alerts).
Добавим в index.html в div.container div с class="p-alert" (p -- placeholder). В него будет рендерится наш компонент.

Добавляем все необходимые зависимости в project.clj:
``` clojure
  :dependencies [[org.clojure/clojure "1.5.1"]
                 [org.clojure/clojurescript "0.0-2173"]
                 [org.clojure/core.async "0.1.278.0-76b25b-alpha"]
                 [quiescent "0.1.1"]]
```
Определим первый компонент в core.cljs:
``` clojure
(ns random-charts.core
  (:require [quiescent :as q :include-macros true]
            [quiescent.dom :as d]))

(q/defcomponent Alert
  [data]
  (if data
    (let [{:keys [text type]} data]
      (d/div {:className (str "alert alert-dismissable alert-" type)}
        (d/button {:type "button" :className "close"} "×")
        text))
    (d/div {} "")))
```
Чтобы REPL продолжил работать, приходится снова открыть Connect bar, отключится от random-charts-project.clj, и заново вычислить (ns random-charts.core (:require...)). Перезагружаем страницу, и вычисление q/defcomponent Alert должно пройти успешно.

Namespace quiescent.dom позволяет нам рендерить dom-элементы через функции d/some-dom-tag. Каждая такая функция принимает на вход map с аргументами и body вторым параметром. Body может быть вызовом еще одного d/some-dom-tag, другого компонента, функции, которая вернет строку, или просто строкой. Например, запись вида (d/span {:className "m-open"} "Hello!") вставит в dom: 
``` html
<span class="m-open">Hello!</span>
```

Наш компонент принимает данные в виде {:text "New alert!" :type "danger"}. Если данных нет, то рендерим пустой div; иначе рендерим div с button и текстом внутри. Вставим наш alert в dom:
``` clojure
(q/render (Alert {:text "New alert!" :type "danger"})
          (.getElementById js/document "alert"))
```
q/render принимает в качестве первого аргумента вызов корневого компонента, а в качестве второго -- dom-элемент, в который будет вставлен компонент. Если в браузере появился alert, значит все работает!
{% img http://snapr.pw/i/69d727af29.png %}
Можно поменять текст или тип предупреждения, заново вычислить q/render, и убедиться, что в браузере все поменялось.
На этом сам quiescent практически весь изучен, но осталось самое интересное -- обновление компонентов. Сам quiescent не занимается state managment'ом, поэтому мы реализуем его сами с помощью core.async.

## Простое обновление компонента

Давайте выведем в наше предупреждение текущее время, и будем его обновлять раз в секунду.
Для начала, проверим, что мы можем получить текущее время и вывести его:
``` clojure
(q/render (Alert {:text (js/Date) :type "success"})
          (.getElementById js/document "alert"))
```
Нам надо как-то 1) Обновлять наше состояние раз в секунду и 2) Уведомлять об этом quiescent, чтобы он перерисовал компонент.
Для этого мы создадим канал, в который будем посылать обновления нашего состояния. Затем подпишемся на канал, и в подписчике будем вызывать перерисовку компонента.

``` clojure
;; Создаем atom для состояния и канал для обновления:
(def app-state (atom {}))
(def update-channel (a/chan))

;; Функция subscribe бесконечно считывает значения из канала, делает swap! с состоянием и вызывает q/render:
(defn subscribe
  [update-fn state ch]
  (am/go (while true
           (let [val (a/<! ch)
                 _ (.log js/console (str "recieved value [" val "]"))
                 new-state (swap! state update-fn val)]
             (q/render 
              (Alert new-state)
              (.getElementById js/document "alert"))))))

;; Подпишемся на наш канал, в функции обновления просто запишем новое значение в :text
;; В нашем случае подошел бы reset!, либо (assoc state :text v), но swap! более универсален, и для примера оставим так.
(subscribe (fn [s v] {:text v :type "success"}) app-state update-channel)

;; Функция будет бесконечно раз в секунду посылать текущую дату в канал:
(defn publish-every-second
  [channel]
  (am/go 
   (loop []
     (a/<! (a/timeout 1000))
     (a/>! channel (js/Date))
     (recur))))

;; Вызываем:
(publish-every-second update-channel)
```
В браузере мы должны увидеть рабочие часы.


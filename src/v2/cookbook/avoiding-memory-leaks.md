---
title: Устранение утечек памяти
type: cookbook
order: 10
---

## Введение

Если вы разрабатываете приложения на Vue, тогда вам нужно следить за утечками памяти. Эта проблема особенно важна в одностраничных приложениях (SPA), потому что идея использования SPA состоит в отсутствии пользователям необходимости обновлять страницы браузера, поэтому задачей JavaScript приложения также будет и очистка в компонентах от лишнего.

Утечки памяти в приложениях Vue обычно не исходят от самого Vue, скорее они могут происходить при интеграции других библиотек в приложение.

## Простой пример

В этом примере показана утечка памяти, вызванная использованием библиотеки [Choices.js](https://github.com/jshjohnson/Choices) внутри компонента Vue без очистки ресурсов должным образом. Далее мы покажем как удалять остающееся после Choices.js и избежать утечки памяти.

В примере ниже, мы загружаем в select большое число вариантов выбора, а также используем кнопку для отображения/скрытия с помощью директивы [v-if](../guide/conditional.html), чтобы добавлять и удалять список из виртуального DOM. Проблема в этом примере заключается в том, что директива `v-if` удаляет родительский элемент из DOM, но не выполняет дополнительную очистку DOM от частей, созданных Choices.js, что и вызывает утечку памяти.

```html
<link rel="stylesheet prefetch" href="https://joshuajohnson.co.uk/Choices/assets/styles/css/choices.min.css?version=3.0.3">
<script src="https://joshuajohnson.co.uk/Choices/assets/scripts/dist/choices.min.js?version=3.0.3"></script>

<div id="app">
  <button
    v-if="showChoices"
    @click="hide"
  >Скрыть</button>
  <button
    v-if="!showChoices"
    @click="show"
  >Показать</button>
  <div v-if="showChoices">
    <select id="choices-single-default"></select>
  </div>
</div>
```

```js
new Vue({
  el: "#app",
  data: function () {
    return {
      showChoices: true
    }
  },
  mounted: function () {
    this.initializeChoices()
  },
  methods: {
    initializeChoices: function () {
      let list = []
      // загружаем в наш select множество вариантов,
      // что вызовет большое использование памяти
      for (let i = 0; i < 1000; i++) {
        list.push({
          label: "Item " + i,
          value: i
        })
      }
      new Choices("#choices-single-default", {
        searchEnabled: true,
        removeItemButton: true,
        choices: list
      })
    },
    show: function () {
      this.showChoices = true
      this.$nextTick(() => {
        this.initializeChoices()
      })
    },
    hide: function () {
      this.showChoices = false
    }
  }
})
```

Чтобы увидеть эту утечку памяти в действии, откройте этот [пример на CodePen](https://codepen.io/freeman-g/pen/qobpxo) с помощью Chrome и затем откройте Диспетчер задач Chrome. Чтобы открыть на Mac, выберите в верхнем меню > Окно > Диспетчер задач или на Windows с помощью сочетания клавиш Shift+Esc. Теперь, нажимайте кнопку показать/скрыть около 50 раз. Вы сможете увидеть увеличение использованной памяти в Диспетчере задач Chrome, которая не будет освобождена.

![Пример утечки памяти](/ru.vuejs.org/images/memory-leak-example.png)

## Исправление утечки памяти

В примере выше, мы можем использовать наш метод `hide()` для выполнения очистки и устранения утечки памяти перед удалением select из DOM. Для этого мы будем хранить свойство в нашем экземпляре Vue и будем использовать [API плагина Choices](https://github.com/jshjohnson/Choices) в методе `destroy()` для выполнения необходимых операций очистки.

Проверьте использование памяти в [обновлённом примере на CodePen](https://codepen.io/freeman-g/pen/mxWMor).

```js
new Vue({
  el: "#app",
  data: function () {
    return {
      showChoices: true,
      choicesSelect: null
    }
  },
  mounted: function () {
    this.initializeChoices()
  },
  methods: {
    initializeChoices: function () {
      let list = []
      for (let i = 0; i < 1000; i++) {
        list.push({
          label: "Item " + i,
          value: i
        })
      }
      // Сохраняем ссылку на экземпляр плагина
      // в объекте data экземпляра Vue
      this.choicesSelect = new Choices("#choices-single-default", {
        searchEnabled: true,
        removeItemButton: true,
        choices: list
      })
    },
    show: function () {
      this.showChoices = true
      this.$nextTick(() => {
        this.initializeChoices()
      })
    },
    hide: function () {
      // теперь мы можем использовать ссылку на Choices
      // для выполнения необходимых плагину операций очистки
      // перед удалением элементов из DOM
      this.choicesSelect.destroy()
      this.showChoices = false
    }
  }
})
```

## Подробнее о значимости

Управлением памятью и тестированием производительности часто легко можно пренебречь в спешке выпустить готовый продукт, но, тем не менее, сохранение небольшого количества используемой памяти по-прежнему важно для пользовательского опыта использования в целом.

Определите типы устройств, которые ваши пользователи могут использовать и какой сценарий работы с приложением может быть. Могут ли они использовать ноутбуки или мобильные устройства с небольшим количеством памяти? Будут ли ваши пользователи обычно совершать много переходов между страницами приложения? Если ответы на эти вопросы — «да», тогда хорошие практики управления памятью могут помочь избежать вам наихудшего сценария сбоя браузера пользователя. Даже если ни один ответ на вопрос не будет «да», вы по-прежнему можете ухудшить производительность вашего приложения при длительном использовании, если не будете осторожны.

## Пример из жизни

В примере выше, мы использовали директиву `v-if` чтобы проиллюстрировать утечку памяти, но более распространённый сценарий из жизни возникает при использовании [vue-router](https://v3.router.vuejs.org/ru/) для маршрутизации по компонентам в одностраничном приложении (SPA).

Также, как и директива `v-if`, `vue-router` удаляет элементы из виртуального DOM и заменяет их новыми элементами при навигации пользователя по вашему приложению. [Хук жизненного цикла](../guide/instance.html#Диаграмма-жизненного-цикла) `beforeDestroy()` — хорошее место для решения подобной проблемы в приложениях на основе `vue-router`.

Мы могли бы переместить нашу очистку в хук `beforeDestroy()` следующим образом:

```js
beforeDestroy: function () {
  this.choicesSelect.destroy()
}
```

## Альтернативы

Мы обсудили управление памятью при удалении элементов, но что, если вы намеренно хотите сохранять состояние и сохранить элементы в памяти? В этом случае вы можете использовать встроенный компонент [keep-alive](/v2/api/#keep-alive).

Когда вы оборачиваете компонент с помощью `keep-alive`, его состояние будет сохранено, и следовательно, останется в памяти.

```html
<button @click="show = false">Скрыть</button>
<keep-alive>
  <!-- my-component будет сохраняться в памяти даже при удалении -->
  <my-component v-if="show"></my-component>
</keep-alive>
```
Эта техника может быть полезна для улучшения пользовательского опыта работы с приложением. Например, представьте, что пользователь начинает вводить комментарий в текстовое поле и затем решает перейти на другую страницу. Если пользователь вернётся обратно, то его комментарий останется сохранён в поле.

С тех пор, как начнёте использовать keep-alive, у вас появится доступ к двум дополнительным хукам жизненного цикла: `activated` и `deactivated`. Если вы хотите выполнить очистку или изменить данные при удалении компонента с keep-alive, вы можете сделать это в хуке `deactivated`.

```js
deactivated: function () {
  // удаление любых данных, которые не требуется хранить
}
```

## Подытожим

Vue позволяет легко разрабатывать потрясающие реактивные JavaScript-приложения, но вам всё равно нужно уделять внимание утечкам памяти. Эти утечки обычно происходят при использовании дополнительных сторонних библиотек, которые манипулируют DOM вне Vue. Не забудьте проверить ваше приложение на утечки памяти и предпринять соответствующие шаги для добавления необходимых очисток в компонентах, где это необходимо.

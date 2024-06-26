---
title: Render-функции и JSX
type: guide
order: 303
---

## Основы

В большинстве случаев для формирования HTML с помощью Vue рекомендуется использовать шаблоны. Впрочем, иногда возникает необходимость в использовании всех алгоритмических возможностей JavaScript. В таких случаях можно применить **`render`-функции** — более низкоуровневую альтернативу шаблонам.

Давайте разберём простой пример, в котором использование `render`-функции будет целесообразным. Предположим, вы хотите сгенерировать заголовки с «якорями»:

```html
<h1>
  <a name="hello-world" href="#hello-world">
    Привет, мир!
  </a>
</h1>
```

Для генерации представленного выше HTML вы решаете использовать такой интерфейс компонента:

```html
<anchored-heading :level="1">Привет, мир!</anchored-heading>
```

При использовании шаблонов для реализации интерфейса, который генерирует только заголовок, основанный на свойстве `level`, вы быстро столкнётесь со следующей ситуацией:

```html
<script type="text/x-template" id="anchored-heading-template">
  <h1 v-if="level === 1">
    <slot></slot>
  </h1>
  <h2 v-if="level === 2">
    <slot></slot>
  </h2>
  <h3 v-if="level === 3">
    <slot></slot>
  </h3>
  <h4 v-if="level === 4">
    <slot></slot>
  </h4>
  <h5 v-if="level === 5">
    <slot></slot>
  </h5>
  <h6 v-if="level === 6">
    <slot></slot>
  </h6>
</script>
```

```js
Vue.component('anchored-heading', {
  template: '#anchored-heading-template',
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

Выглядит не очень. Шаблон получился не только очень большим — приходится ещё и `<slot></slot>` повторять для каждого возможного уровня заголовка.

Шаблоны хорошо подходят для большинства компонентов, но рассматриваемый сейчас — явно не один из них. Давайте попробуем переписать компонент, используя `render`-функцию:

```js
Vue.component('anchored-heading', {
  render: function (createElement) {
    return createElement(
      'h' + this.level,   // имя тега
      this.$slots.default // массив дочерних элементов
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

Так-то лучше, наверное? Код короче, но требует более подробного знакомства со свойствами экземпляра Vue. В данном случае необходимо знать, что когда дочерние элементы передаются без указания директивы `v-slot`, как например `Привет, мир!` внутри `anchored-heading`, они сохраняются в экземпляре компонента как `$slots.default`. Если вы этого ещё не сделали, **советуем вам пробежать глазами [API-справочник свойств экземпляра](../api/#Свойства-экземпляра), перед тем как углубляться в рассмотрение render-функций.**

## Узлы, деревья, и виртуальный DOM

Прежде чем погрузиться в `render`-функции, необходимо узнать, как работают браузеры. Возьмём, например, этот HTML-код:

```html
<div>
  <h1>My title</h1>
  Some text content
  <!-- TODO: Add tagline -->
</div>
```

Когда браузер обрабатывает этот код, он создаёт [дерево «узлов DOM»](https://learn.javascript.ru/dom-nodes), чтобы облегчить ему отслеживание всего, как, например, вы могли бы построить генеалогическое дерево для отслеживания вашей увеличивающейся семьи.

Дерево узлов DOM для HTML из примера выше выглядит следующим образом:

![Визуализация дерева DOM](/ru.vuejs.org/images/dom-tree.png)

Каждый элемент является узлом. Каждый фрагмент текста является узлом. Даже комментарии это узлы! Узел — это всего лишь часть страницы. И так же, как и в генеалогическом дереве, каждый узел может иметь своих потомков (т.е. каждая часть может содержать в себе другие части).

Обновление всех этих узлов может быть затруднительно, но, к счастью, вам не придётся делать это вручную. Вы просто вставляете в шаблон определённый HTML-код, который вы хотите отобразить на странице:

```html
<h1>{{ blogTitle }}</h1>
```

Или то же, но с использованием render-функции:

```js
render: function (createElement) {
  return createElement('h1', this.blogTitle)
}
```

В обоих случаях Vue автоматически будет обновлять страницу при изменениях `blogTitle`.

### Виртуальный DOM

Vue реализует это созданием **виртуального DOM**, чтобы отслеживать изменения, которые ему требуется внести в реальный DOM. Рассмотрим подробнее следующую строку:

```js
return createElement('h1', this.blogTitle)
```

Что в действительности возвращает `createElement`? Это не _в точности_ реальный DOM-элемент. Можно было бы назвать `createNodeDescription`, если быть точным, так как результат содержит информацию, описывающую Vue, какой именно узел должен быть отображён на странице, включая описания любых дочерних узлов. Мы называем это описание узла «виртуальным узлом», обычно сокращая до аббревиатуры **VNode**. «Виртуальный DOM» — это то, что мы называем всем деревом VNodes, созданным из дерева компонентов Vue.

## Аргументы `createElement`

Следующий момент, с которым необходимо познакомиться, — это синтаксис использования возможностей шаблонизации функцией `createElement`. Вот аргументы, которые принимает `createElement`:

```js
// @returns {VNode}
createElement(
  // {String | Object | Function}
  // Название тега HTML, опции компонента или асинхронная функция,
  // возвращающая один из двух них. Обязательный параметр.
  'div',

  // {Object}
  // Объект данных, содержащий атрибуты,
  // который вы бы указали в шаблоне. Опциональный параметр.
  {
    // (см. детали в секции ниже)
  },

  // {String | Array}
  // Дочерние виртуальные узлы (VNode), создаваемые с помощью `createElement()`
  // или просто строки для получения 'текстовых VNode'. Опциональный параметр.
  [
    'Какой-то текст прежде всех остальных элементов.',
    createElement('h1', 'Заголовок'),
    createElement(MyComponent, {
      props: {
        someProp: 'foobar'
      }
    })
  ]
)
```

### Подробно об объекте данных

Обратите внимание: особым образом рассматриваемые в шаблонах атрибуты `v-bind:class` и `v-bind:style`, и в объектах данных виртуальных узлов имеют собственные поля на верхнем уровне объектов данных. Этот объект также позволяет вам связывать обычные атрибуты HTML, а также свойства DOM, такие как `innerHTML` (это заменит директиву `v-html`):

```js
{
  // То же API, что и у `v-bind:class`, принимающий
  // строку, объект, массив строк или массив объектов
  class: {
    foo: true,
    bar: false
  },
  // То же API, что и у `v-bind:style`, принимающий
  // строку, объект, массив строк или массив объектов
  style: {
    color: 'red',
    fontSize: '14px'
  },
  // Обычные атрибуты HTML
  attrs: {
    id: 'foo'
  },
  // Входные параметры компонентов
  props: {
    myProp: 'bar'
  },
  // Свойства DOM
  domProps: {
    innerHTML: 'baz'
  },
  // Обработчики событий располагаются под ключом `on`,
  // однако модификаторы, вроде как `v-on:keyup.enter`, не
  // поддерживаются. Проверять keyCode придётся вручную.
  on: {
    click: this.clickHandler
  },
  // Только для компонентов. Позволяет слушать нативные события,
  // а не генерируемые в компоненте через `vm.$emit`.
  nativeOn: {
    click: this.nativeClickHandler
  },
  // Пользовательские директивы. Обратите внимание, что `oldValue`
  // не может быть указано, так как Vue сам его отслеживает
  directives: [
    {
      name: 'my-custom-directive',
      value: '2',
      expression: '1 + 1',
      arg: 'foo',
      modifiers: {
        bar: true
      }
    }
  ],
  // Слоты с ограниченной областью видимости в формате
  // { name: props => VNode | Array<VNode> }
  scopedSlots: {
    default: props => createElement('span', props.text)
  },
  // Имя слота, если этот компонент
  // является потомком другого компонента
  slot: 'name-of-slot',
  // Прочие специальные свойства верхнего уровня
  key: 'myKey',
  ref: 'myRef',
  // Если указываете одно имя в ref к нескольким элементам
  // в render-функции — это сделает `$refs.myRef` массивом
  refInFor: true
}
```

### Полный пример

Узнав всё это, мы теперь можем завершить начатый ранее компонент:

```js
var getChildrenTextContent = function (children) {
  return children.map(function (node) {
    return node.children
      ? getChildrenTextContent(node.children)
      : node.text
  }).join('')
}

Vue.component('anchored-heading', {
  render: function (createElement) {
    // создать id в kebab-case
    var headingId = getChildrenTextContent(this.$slots.default)
      .toLowerCase()
      .replace(/\W+/g, '-')
      .replace(/(^-|-$)/g, '')

    return createElement(
      'h' + this.level,
      [
        createElement('a', {
          attrs: {
            name: headingId,
            href: '#' + headingId
          }
        }, this.$slots.default)
      ]
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

### Ограничения

#### Виртуальные узлы должны быть уникальными

Все виртуальные узлы в компоненте должны быть уникальными. Это значит, что `render`-функция ниже некорректна:

```js
render: function (createElement) {
  var myParagraphVNode = createElement('p', 'hi')
  return createElement('div', [
    // Упс — дублирующиеся виртуальные узлы!
    myParagraphVNode, myParagraphVNode
  ])
}
```

Если вы действительно хотите многократно использовать один и тот же элемент/компонент, примените функцию-фабрику. Например, следующая `render`-функция полностью корректный способ для отображения 20 одинаковых абзацев:

```js
render: function (createElement) {
  return createElement('div',
    Array.apply(null, { length: 20 }).map(function () {
      return createElement('p', 'Привет')
    })
  )
}
```

## Реализация возможностей шаблона с помощью JavaScript

### `v-if` и `v-for`

Функциональность, легко реализуемая в JavaScript, не требует от Vue какой-либо проприетарной альтернативы. Например, используемые в шаблонах `v-if` и `v-for`:

```html
<ul v-if="items.length">
  <li v-for="item in items">{{ item.name }}</li>
</ul>
<p v-else>Ничего не найдено.</p>
```

При использовании `render`-функции это можно легко переписать с помощью `if`/`else` и `map`:

```js
props: ['items'],
render: function (createElement) {
  if (this.items.length) {
    return createElement('ul', this.items.map(function (item) {
      return createElement('li', item.name)
    }))
  } else {
    return createElement('p', 'Ничего не найдено.')
  }
}
```

### `v-model`

В `render`-функции нет прямого аналога `v-model` — вы должны реализовать эту логику самостоятельно:

```js
props: ['value'],
render: function (createElement) {
  var self = this
  return createElement('input', {
    domProps: {
      value: self.value
    },
    on: {
      input: function (event) {
        self.$emit('input', event.target.value)
      }
    }
  })
}
```

Это цена использования низкоуровневой реализации, которая в то же время предоставляет вам больше контроля над взаимодействием, чем `v-model`.

### События и модификаторы клавиш

Для модификаторов событий `.passive`, `.capture` и `.once`, Vue предоставляет префиксы, которые могут быть использованы вместе с `on`:

| Модификаторы | Префикс |
| ------ | ------ |
| `.passive` | `&` |
| `.capture` | `!` |
| `.once` | `~` |
| `.capture.once` или<br>`.once.capture` | `~!` |

Например:

```javascript
on: {
  '!click': this.doThisInCapturingMode,
  '~keyup': this.doThisOnce,
  '~!mouseover': this.doThisOnceInCapturingMode
}
```

Для всех остальных событий и модификаторов клавиш нет необходимости в префиксе, потому что вы можете просто использовать методы события в обработчике:

| Модификаторы | Эквивалент в обработчике |
| ------ | ------ |
| `.stop` | `event.stopPropagation()` |
| `.prevent` | `event.preventDefault()` |
| `.self` | `if (event.target !== event.currentTarget) return` |
| Клавиши:<br>`.enter`, `.13` | `if (event.keyCode !== 13) return` (измените `13` на [любой другой код клавиши](http://keycode.info/) для модификаторов других клавиш) |
| Модификаторы клавиш:<br>`.ctrl`, `.alt`, `.shift`, `.meta` | `if (!event.ctrlKey) return` (измените `ctrlKey` на `altKey`, `shiftKey` или `metaKey` соответственно) |

Пример использования всех этих модификаторов вместе:

```javascript
on: {
  keyup: function (event) {
    // Ничего не делаем, если элемент на котором произошло
    // событие не является элементом, который мы отслеживаем
    if (event.target !== event.currentTarget) return

    // Ничего не делаем, если клавиша не является Enter (13)
    // и клавиша SHIFT не была нажата в то же время
    if (!event.shiftKey || event.keyCode !== 13) return

    // Останавливаем всплытие события
    event.stopPropagation()

    // Останавливаем стандартный обработчик keyup для этого элемента
    event.preventDefault()
    // ...
  }
}
```

### Слоты

Вы можете получить доступ к статическому содержимому слотов в виде массивов VNode, используя [`this.$slots`](../api/#vm-slots):

```js
render: function (createElement) {
  // `<div><slot></slot></div>`
  return createElement('div', this.$slots.default)
}
```

И получить доступ к слотам со своей областью видимости как к функциям, возвращающим VNode, используя [`this.$scopedSlots`](../api/#vm-scopedSlots):

```js
props: ['message'],
render: function (createElement) {
  // `<div><slot :text="message"></slot></div>`
  return createElement('div', [
    this.$scopedSlots.default({
      text: this.message
    })
  ])
}
```

Чтобы передать слоты со своей областью видимости в дочерний компонент при помощи `render`-функции, добавьте свойство `scopedSlots` в данные VNode:

```js
render: function (createElement) {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return createElement('div', [
    createElement('child', {
      // передаём `scopedSlots` в объект data
      // в виде { name: props => VNode | Array<VNode> }
      scopedSlots: {
        default: function (props) {
          return createElement('span', props.text)
        }
      }
    })
  ])
}
```

## JSX

Если приходится писать много `render`-функций, то такой код может утомлять:

```js
createElement(
  'anchored-heading', {
    props: {
      level: 1
    }
  }, [
    createElement('span', 'Привет'),
    ', мир!'
  ]
)
```

Особенно в сравнении с кодом аналогичного шаблона:

```html
<anchored-heading :level="1">
  <span>Привет</span>, мир!
</anchored-heading>
```

Поэтому есть [плагин для Babel](https://github.com/vuejs/jsx), позволяющий использовать JSX во Vue, и применять синтаксис, похожий на шаблоны:

```js
import AnchoredHeading from './AnchoredHeading.vue'

new Vue({
  el: '#demo',
  render: function (h) {
    return (
      <AnchoredHeading level={1}>
        <span>Привет</span>, мир!
      </AnchoredHeading>
    )
  }
})
```

<p class="tip">Сокращение `createElement` до `h` — распространённое соглашение в экосистеме Vue и обязательное для использования JSX. Начиная с [версии 3.4.0](https://github.com/vuejs/babel-plugin-transform-vue-jsx#h-auto-injection) плагина Babel для Vue, мы автоматически внедряем `const h = this.$createElement` в любой метод и геттер (не функциях или стрелочных функциях), объявленных в синтаксисе ES2015 с JSX, поэтому можно удалить параметр `(h)`. В предыдущих версиях плагина, ваше приложение будет выкидывать ошибку, если `h` недоступен в области видимости.</p>

Подробную информацию о преобразовании JSX в JavaScript можно найти в [документации плагина](https://github.com/vuejs/jsx#installation).

## Функциональные компоненты

Компонент для заголовков с «якорями», который мы создали выше, довольно прост. У него нет какого-либо состояния, хуков или требующих наблюдения данных. По сути, это всего лишь функция с параметром.

В подобных случаях мы можем пометить компоненты как функциональные (опция `functional`), что означает отсутствие у них состояния (нет [реактивных данных](../api/#Опции-—-данные)) и экземпляра (нет переменной контекста `this`). **Функциональный компонент** выглядит так:

```js
Vue.component('my-component', {
  functional: true,
  // входные параметры опциональны
  props: {
    // ...
  },
  // чтобы компенсировать отсутствие экземпляра
  // мы передаём контекст вторым аргументом
  render: function (createElement, context) {
    // ...
  }
})
```

> Примечание: в версиях до 2.3.0 опция `props` необходима, если вы хотите использовать входные параметры в функциональном компоненте. С версии 2.3.0 и выше вы можете опустить опцию `props` и все атрибуты, найденные на узле компонента, будут неявно извлечены в качестве входных данных.
>
> Ссылка будет указывать на HTMLElement при использовании с функциональными компонентами, потому что у них нет состояния и экземпляра.

С версии 2.5.0+, если вы используете [однофайловые компоненты](single-file-components.html), вы можете объявить функциональные компоненты, основанные на шаблоне таким образом:

```html
<template functional>
</template>
```

Всё необходимое компоненту передаётся через `context` — объект, содержащий следующие поля:

- `props`: Объект со всеми переданными входными параметрами
- `children`: Массив дочерних виртуальных узлов
- `slots`: Функция, возвращающая объект со слотами
- `scopedSlots`: (2.6.0+) Объект, содержащий все переданные слоты с ограниченной областью видимости. Также предоставляет доступ к обычным слотам в качестве функций
- `data`: [Объект данных](#Подробно-об-объекте-данных) целиком, переданный объекту вторым аргументом `createElement`
- `parent`: Ссылка на родительский компонент
- `listeners`: (2.3.0+) Объект, содержащий все зарегистрированные в родителе прослушиватели событий. Это просто псевдоним для `data.on`
- `injections`: (2.3.0+) Если используется опция [`inject`](../api/#provide-inject), будет содержать все разрешённые инъекции.

После указания `functional: true`, обновление `render`-функции нашего компонента для заголовков потребует только добавления параметра `context`, обновления `this.$slots.default` на `context.children` и замены `this.level` на `context.props.level`.

Поскольку функциональные компоненты — это просто функции, их отрисовка значительно быстрее.

Кроме того, они очень удобны в качестве обёрток. Например, если вам нужно:

- Выбрать один из компонентов для последующей отрисовки в данной точке
- Произвести манипуляции над дочерними элементами, входными параметрами или данными, перед тем как передать их в дочерний компонент

Вот пример компонента `smart-list`, делегирующего отрисовку к более специализированным компонентам, в зависимости от переданных в него данных:

```js
var EmptyList = { /* ... */ }
var TableList = { /* ... */ }
var OrderedList = { /* ... */ }
var UnorderedList = { /* ... */ }

Vue.component('smart-list', {
  functional: true,
  props: {
    items: {
      type: Array,
      required: true
    },
    isOrdered: Boolean
  },
  render: function (createElement, context) {
    function appropriateListComponent () {
      var items = context.props.items

      if (items.length === 0)           return EmptyList
      if (typeof items[0] === 'object') return TableList
      if (context.props.isOrdered)      return OrderedList

      return UnorderedList
    }

    return createElement(
      appropriateListComponent(),
      context.data,
      context.children
    )
  }
})
```

### Передача атрибутов и событий дочерним элементам/компонентам

В обычных компонентах, атрибуты не определённые как входные параметры, автоматически добавляются к корневому элементу компонента, заменяя или [правильно объединяя](class-and-style.html) любые существующие атрибуты с тем же именем.

Однако функциональные компоненты требуют явного определения этого поведения:

```js
Vue.component('my-functional-button', {
  functional: true,
  render: function (createElement, context) {
    // Явная передача любых атрибутов, слушателей событий, дочерних элементов и т.д.
    return createElement('button', context.data, context.children)
  }
})
```

Передавая `context.data` вторым аргументом в `createElement`, мы передаём любые атрибуты или слушатели событий, используемые в `my-functional-button`. На самом деле это настолько очевидно, что для событий не требуется модификатор `.native`.

Если вы используете функциональные компоненты на основе шаблонов, вам также придётся вручную добавлять атрибуты и слушатели. Поскольку у нас есть доступ к индивидуальному содержимому контекста, мы можем использовать `data.attrs` для передачи любых атрибутов HTML и `listeners` _(псевдоним для `data.on`)_ для передачи любых слушателей событий.

```html
<template functional>
  <button
    class="btn btn-primary"
    v-bind="data.attrs"
    v-on="listeners"
  >
    <slot/>
  </button>
</template>
```

### `slots()` vs `children`

Вы можете задаться вопросом зачем нужны `slots()` и `children` одновременно. Разве не будет `slots().default` возвращать тот же результат, что и `children`? В некоторых случаях — да, но что если у нашего функционального компонента будут следующие дочерние элементы?

```html
<my-functional-component>
  <p v-slot:foo>
    первый
  </p>
  <p>второй</p>
</my-functional-component>
```

Для этого компонента, `children` даст вам оба абзаца, `slots().default` — только второй, а `slots().foo` — только первый. Таким образом, наличие и `children`, и `slots()` позволяет выбрать, знает ли компонент о системе слотов или просто делегировать это другому компоненту, путём передачи `children`.

## Компиляция шаблонов

Возможно, вам будет интересно узнать, что Vue-шаблоны в действительности компилируются в `render`-функцию. Обычно нет необходимости знать подобные детали реализации, но может быть любопытным посмотреть на то, как компилируются те или иные возможности шаблонов. Ниже приведена небольшая демонстрация использования метода `Vue.compile`, который в режиме реального времени компилирует строки шаблонов:

<iframe src="https://codesandbox.io/embed/github/vuejs/vuejs.org/tree/master/src/v2/examples/vue-20-template-compilation?codemirror=1&hidedevtools=1&hidenavigation=1&theme=light&view=preview" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" title="vue-20-template-compilation" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

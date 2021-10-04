---
sidebarDepth: 1
---

# SFC и синтаксис `<script setup>`

`<script setup>` — синтаксический сахар, обрабатываемый на этапе компиляции, для использования [Composition API](composition-api.md) в однофайловых компонентах (SFC). Это рекомендуемый синтаксис при использовании однофайловых компонентов и Composition API. Он предлагает ряд преимуществ по сравнению с обычным синтаксисом `<script>`:

- Более лаконичный код с меньшим количеством boilerplate-кода
- Возможность объявлять входные параметры и генерируемые события с использованием чистого TypeScript
- Лучшая производительность во время выполнения (шаблон компилируется в render-функцию в той же области видимости, без промежуточной прокси)
- Лучшая производительность IDE при определении типов (меньше работы для языкового сервера по извлечению типов из кода)

## Базовый синтаксис

Чтобы использовать синтаксис, добавьте атрибут `setup` в секцию `<script>`:

```vue
<script setup>
console.log('привет синтаксис script setup')
</script>
```

Код внутри компилируется как содержимое функции компонента `setup()`. Это означает, что в отличие от обычного `<script>`, который выполняется только один раз при первом импорте компонента, код внутри `<script setup>` будет **выполняться каждый раз при создании экземпляра компонента**.

### Привязки верхнего уровня будут доступны в шаблоне

При использовании `<script setup>` любые привязки верхнего уровня (в т.ч. переменные, объявления функций и импорты) объявленные внутри `<script setup>` будут доступны напрямую в шаблоне:

```vue
<script setup>
// переменная
const msg = 'Hello!'

// функция
function log() {
  console.log(msg)
}
</script>

<template>
  <div @click="log">{{ msg }}</div>
</template>
```

Импорты объявляются таким же образом. Это означает, что можно напрямую использовать импортированную вспомогательную функцию в выражениях шаблона, без необходимости объявлять её через опцию `methods`:

```vue
<script setup>
import { capitalize } from './helpers'
</script>

<template>
  <div>{{ capitalize('hello') }}</div>
</template>
```

## Реактивность

Реактивное состояние нужно явно создавать с помощью [API реактивности](basic-reactivity.md). Аналогично значениям, возвращаемым из функции `setup()`, ref-ссылки автоматически разворачиваются, когда на них ссылаются в шаблонах:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

## Использование компонентов

Значения в области видимости `<script setup>` также могут быть использованы непосредственно в качестве имён тегов пользовательских компонентов:

```vue
<script setup>
import MyComponent from './MyComponent.vue'
</script>

<template>
  <MyComponent />
</template>
```

Считайте, что на `MyComponent` ссылаются как на переменную. Если использовали JSX, то ментальная модель тут аналогична. Эквивалент в kebab-case `<my-component>` работает и в шаблоне — но настоятельно рекомендуем писать теги в PascalCase для консистентности. Это также помогает отличить их от нативных пользовательских элементов.

### Динамические компоненты

Поскольку на компоненты ссылаются как на переменные, а не регистрируют их под строковыми ключами, то внутри `<script setup>` при использовании динамических компонентов потребуется использовать динамическую привязку с помощью `:is`:

```vue
<script setup>
import Foo from './Foo.vue'
import Bar from './Bar.vue'
</script>

<template>
  <component :is="Foo" />
  <component :is="someCondition ? Foo : Bar" />
</template>
```

Обратите внимание, как компоненты могут использоваться в качестве переменных в тернарном выражении.

### Рекурсивные компоненты

Однофайловые компоненты могут неявно ссылаться сами на себя с помощью имени файла. Например, файл с именем `FooBar.vue` может ссылаться на себя как `<FooBar/>` в своём шаблоне.

Обратите внимание, что это имеет более низкий приоритет, чем у импортированных компонентов. Если есть именованный импорт, который конфликтует с предполагаемым именем компонента от имени файла, то можно задать псевдоним для импортируемого:

```js
import { FooBar as FooBarChild } from './components'
```

### Компоненты с пространством имён

Можно использовать теги компонентов с точками, например `<Foo.Bar>`, чтобы ссылаться на компоненты, вложенные в свойства объекта. Это полезно при импорте нескольких компонентов из одного файла:

```vue
<script setup>
import * as Form from './form-components'
</script>

<template>
  <Form.Input>
    <Form.Label>label</Form.Label>
  </Form.Input>
</template>
```

## `defineProps` и `defineEmits`

Чтобы объявить `props` и `emits` при использовании `<script setup>` нужно использовать API `defineProps` и `defineEmits`, которые предоставляют полную поддержку вывода типов и автоматически доступны внутри `<script setup>`:

```vue
<script setup>
const props = defineProps({
  foo: String
})

const emit = defineEmits(['change', 'delete'])

// код setup
</script>
```

- `defineProps` и `defineEmits` — **макросы компилятора** используемые только внутри `<script setup>`. Их не нужно импортировать и они будут компилироваться при обработке `<script setup>`.

- `defineProps` принимает то же значение, что и [опция `props`](options-data.md#props), а `defineEmits` принимает то же значение, что и [опция `emits`](options-data.md#emits).

- `defineProps` и `defineEmits` предоставляют правильный вывод типов на основе переданных опций.

- Опции, переданные в `defineProps` и `defineEmits` будут подняты из setup в область видимости модуля. Поэтому опции не могут ссылаться на локальные переменные, объявленные в области видимости setup. Это приведёт к ошибке компиляции. Однако, они _могут ссылаться_ на импортированные привязки, поскольку они также находятся в области видимости модуля.

При использовании TypeScript можно также [объявлять входные параметры и события с помощью аннотации чистых типов](#возможности-только-для-typescript).

## `defineExpose`

Компоненты со `<script setup>` **по умолчанию закрытые** — т.е. публичный экземпляр компонента, получаемый через ссылку в шаблоне или цепочку `$parent`, **не объявляет** доступа к каким-либо привязкам внутри `<script setup>`.

Чтобы явно объявить свойства в компоненте со `<script setup>`, воспользуйтесь макросом компилятора `defineExpose`:

```vue
<script setup>
import { ref } from 'vue'

const a = 1
const b = ref(2)

defineExpose({
  a,
  b
})
</script>
```

Когда родитель запросит экземпляр этого компонента через ссылку в шаблоне, то полученный экземпляр будет иметь вид `{ a: number, b: number }` (ref-ссылки автоматически разворачиваются, как и для обычных экземпляров).

## `useSlots` и `useAttrs`

Использование `slots` и `attrs` внутри `<script setup>` должно встречаться крайне редко, так как в шаблоне прямой доступ к ним можно получить через `$slots` и `$attrs`. В редких случаях, когда они всё же нужны, можно воспользоваться вспомогательными методами `useSlots` и `useAttrs` соответственно:

```vue
<script setup>
import { useSlots, useAttrs } from 'vue'

const slots = useSlots()
const attrs = useAttrs()
</script>
```

`useSlots` и `useAttrs` — фактически будут runtime-функциями, которые возвращают эквивалент `setupContext.slots` и `setupContext.attrs`. Они также могут быть использованы в обычных функциях composition API.

## Использование вместе с обычной секцией `<script>`

`<script setup>` можно использовать и вместе с обычной секцией `<script>`. Обычный `<script>` может понадобиться в случаях, когда необходимо:

- Объявление опций, которые не могут быть выражены в `<script setup>`, например `inheritAttrs` или пользовательские опции, добавляемые плагинами.
- Объявление именованных экспортов.
- Запуск побочных эффекты или создание объектов, которые должны выполняться только один раз.

```vue
<script>
// обычный <script>, выполняется в области видимости модуля (только один раз)
runSideEffectOnce()

// объявление дополнительных опций
export default {
  inheritAttrs: false,
  customOptions: {}
}
</script>

<script setup>
// выполняется в области видимости setup() (для каждого экземпляра)
</script>
```

## Верхне-уровневый `await`

Верхне-уровневый `await` можно использовать внутри `<script setup>`. Полученный код будет скомпилирован как `async setup()`:

```vue
<script setup>
const post = await fetch(`/api/post/1`).then(r => r.json())
</script>
```

Кроме того, ожидаемое выражение будет автоматически скомпилировано в формат, сохраняющий контекст текущего экземпляра компонента после `await`.

:::warning Примечание
`async setup()` нужно использовать в сочетании с `Suspense`, которая в настоящее время всё ещё является экспериментальной функцией. Её планируется доработать и документировать в одном из будущих релизов — но, если интересно, можно изучить её [тесты](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/__tests__/components/Suspense.spec.ts), чтобы увидеть как она работает.
:::

## Возможности только для TypeScript

### Объявление типов для входных параметров/генерируемых событий

Типы входных параметров и генерируемых событий также можно объявить с помощью синтаксиса с передачей аргумента литерального типа в `defineProps` или `defineEmits`:

```ts
const props = defineProps<{
  foo: string
  bar?: number
}>()

const emit = defineEmits<{
  (e: 'change', id: number): void
  (e: 'update', value: string): void
}>()
```

- `defineProps` или `defineEmits` могут использовать только объявления во время runtime ИЛИ объявление типа. Использование обоих одновременно приведёт к ошибке компиляции.

- При использовании объявления типа, эквивалент объявления в runtime автоматически генерируется на основе статического анализа для устранения необходимости двойного объявления и обеспечении корректного поведения во время выполнения.

  - В режиме разработки компилятор попытается вывести из типов соответствующую валидацию в runtime. Например, `foo: String` выводится из типа `foo: string`. Если тип будет ссылкой на импортированный тип, то результатом выведения будет `foo: null` (аналогичный типу `any`), так как компилятор не имеет информации о внешних файлах.

  - В режиме production компилятор сгенерирует массив форматов объявлений для уменьшения размера сборки (входные параметры здесь будут скомпилированы в `['foo', 'bar']`)

  - Выдаваемый код по-прежнему останется TypeScript с правильной типизацией, который может обрабатываться другими инструментами.

- В настоящее время для обеспечения корректного статического анализа, аргумент объявления типа должен быть одним из следующих:

  - Литерал типа
  - Ссылка на интерфейс или литерал типа в том же файле

  В настоящее время сложные типы и импорт типов из других файлов не поддерживается. Теоретически возможно что такая поддержка импортов появится в будущем.

### Значения входных параметров по умолчанию при использовании объявления типов

Один из недостатков объявления типов через `defineProps` в том, что нет возможности указать значения по умолчанию для входных параметров. Для решения этой проблемы создан макрос для компилятора `withDefaults`:

```ts
interface Props {
  msg?: string
  labels?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  msg: 'привет',
  labels: () => ['один', 'два']
})
```

Это скомпилируется в эквиваленты опций `default` в runtime. Кроме того, `withDefaults` предоставляет проверку типов для значений по умолчанию и гарантирует, что возвращаемый тип `props` будет с удалёнными опциональными флагами для свойств, у которых указаны значения по умолчанию.

## Ограничение: Никаких импортов с помощью src

Из-за разницы в семантике выполнения модулей код внутри `<script setup>` полагается на контекст однофайлового компонента. При перемещении во внешние файлы `.js` или `.ts`, это может привести к путанице как для разработчиков, так и для инструментов. Поэтому **`<script setup>`** нельзя использовать с атрибутом `src`.
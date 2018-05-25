# Кэширование

Несмотря на то, что серверный рендеринг в Vue достаточно быстр, он не может сравниться с производительностью шаблонов на основе чистых строк из-за затрат при создании экземпляров компонента и узлов виртуального DOM. В случаях, когда производительность критична, разумное использование стратегий кэширования поможет значительно улучшить время отклика и снизить нагрузку на сервер.

## Кэширование на уровне страниц

Приложение с рендерингом на стороне сервера в большинстве случаев зависит от внешних данных, поэтому содержимое является динамическим, по своей природе, и не может кэшироваться в течение длительных периодов времени. Однако, если содержимое не зависит от конкретного пользователя (например, для одного и того же URL всегда отображается одинаковое содержимое всем пользователям), мы можем использовать стратегию под названием [микро-кэширование](https://www.nginx.com/blog/benefits-of-microcaching-nginx/), чтобы значительно улучшить возможности нашего приложения для обработки большого трафика.

Это обычно реализуется на уровне Nginx, но мы также можем реализовать его в Node.js:

``` js
const microCache = LRU({
  max: 100,
  maxAge: 1000 // Важно: записи устаревают через 1 секунду.
})

const isCacheable = req => {
  // реализация логики проверки является ли запрос зависимым от пользователя.
  // только не зависящие от пользователей запросы должны быть кэшируемыми
}

server.get('*', (req, res) => {
  const cacheable = isCacheable(req)
  if (cacheable) {
    const hit = microCache.get(req.url)
    if (hit) {
      return res.end(hit)
    }
  }

  renderer.renderToString((err, html) => {
    res.end(html)
    if (cacheable) {
      microCache.set(req.url, html)
    }
  })
})
```

Поскольку содержимое кэшируется в течение одной секунды, пользователи не будут видеть устаревший контент. Тем не менее, это означает, что сервер будет выполнять не более одного полного рендеринга в секунду для каждой кэшированной страницы.

## Кэширование на уровне компонентов

`vue-server-renderer` имеет встроенную поддержку кэширования на уровне компонентов. Для её использования вам необходимо предоставить [реализацию кэширования](./api.md#cache) при создании рендерера. Для обычного использования достаточно передать [lru-cache](https://github.com/isaacs/node-lru-cache):

``` js
const LRU = require('lru-cache')

const renderer = createRenderer({
  cache: LRU({
    max: 10000,
    maxAge: ...
  })
})
```

Затем вы можете кэшировать компонент, реализовав функцию `serverCacheKey`:

``` js
export default {
  name: 'item', // требуется
  props: ['item'],
  serverCacheKey: props => props.item.id,
  render (h) {
    return h('div', this.item.id)
  }
}
```

Обратите внимание, что подлежащий кэшированию компонент **также должен определять уникальную опцию `name`**. С уникальным именем ключ кэша таким образом является компоненто-зависимым: вам не нужно беспокоиться о двух компонентах, возвращающих одинаковый ключ.

Ключ, возвращаемый из `serverCacheKey` должен содержать достаточную информацию для представления формы результата рендеринга. Указанное выше является хорошей реализацией, если результат рендеринга определяется исключительно с помощью `props.item.id`. Однако, если элемент с таким же идентификатором может со временем меняться или результат рендеринга также зависит от других данных, вам необходимо изменить реализацию `serverCacheKey`, чтобы учитывать и другие переменные.

При возвращении константы компонент всегда будет кэшироваться, что отлично подходит для чисто статических компонентов.

### Когда использовать кэширование компонентов

Если рендерер найдёт попадание в кэше для компонента во время рендеринга, он будет напрямую переиспользовать кэшированный результат для всего поддерева. Это означает, что вы **НЕ ДОЛЖНЫ** кэшировать компонент когда:

- Он имеет дочерние компоненты, которые могут полагаться на глобальное состояние.
- Он имеет дочерние компоненты, которые создают побочные эффекты (side effects) для `context` рендера.

Поэтому кэширование компонентов следует с аккуратностью применять для устранения узких мест производительности. В большинстве случаев не следует кэшировать компоненты, используемые в единственном экземпляре. Наиболее общий тип компонентов, подходящих для кэширования — те, что повторяются в больших `v-for` циклах. Поскольку такие компоненты, как правило, отражают объекты из коллекции базы данных, они могут использовать простую стратегию кэширования: сгенерируйте их ключи кэша, используя их уникальный идентификатор + временную метку последнего обновления:

``` js
serverCacheKey: props => props.item.id + '::' + props.item.last_updated
```
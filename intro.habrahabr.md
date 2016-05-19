Язык описания шаблонов Snakeskin
================================

Всем привет! В этом посте я хочу рассказать о своей разработке: ЯП для описания тектовых шаблонов - Snakeskin.
Проекту уже более 3.5 лет, и мне кажется, что все детские болезни он пережил, поэтому я могу поделиться результатом с сообществом.

[GitHub](https://github.com/SnakeskinTpl/Snakeskin)

[Документация](http://snakeskintpl.github.io/docs/index-ru.html) (английская версия пока не сделана)

[Смежные проекты](https://github.com/SnakeskinTpl)

## Что, зачем и почему

Так что же за зверь такой этот Snakeskin (далее SS) и зачем я его написал? Мне нравится определение, что SS - это как CoffeeScript или TypeScript, 
т.е. просто "сахарный" синтаксис для JavaScript + дополнительные приятные фичи, которые помогают писать код. 
Но в отличии от этих языков, SS имеет явную специализацию - это описание шаблонов, т.е. конечно можно писать на SS приложения целиком,
но это просто будет не удобно. SS используется не вместо, а вместе с основным языком, например:

**select.ss**

```
- namespace select
- template main(options)
  < select 
    - forEach options => el
      < option value = ${el.value} 
        {el.label}
```

**select.js**

```js
import { select } from 'select.ss';

class Select {
  constructor() {
    this.template = select.main;
  }
}
```

Тут мы описали некоторый класс на JS (ES2015) и подключили как модуль файл написанный на Snakeskin 
(такую интеграцию делает плагин для WebPack), а далее ставим шаблон *main*, как метод класса.

Теперь можно сделать некоторые выводы:

* SS транслируется в JS, который потом бесшовно используется с основным кодом;
* SS полностью сконцентрирован на генерации шаблонов, но в принципе всё, что можно написать на JS - можно написать и на SS.

Зачем же я написал Snakeskin? Дело в том, что мне очень хотелось иметь один язык шаблонов с мощной системой code-reuse как на сервере,
так и на клиенте, причём на клиенте я использую фреймворк (в данный момент Vue), который, как правило, имеет свой язык шаблонов и SS обязан с ним
бесшовно интегрироваться.

## use-case

Теперь рассмотрим основные use-case для Snakeskin:

* Серверная шаблонизация: тут всё просто, подключаем SS шаблон как модуль node.js и работает с его функциями, например:

```js
'use strict';

const http = require('http');
const ss = require('snakeskin');

/// Компилируем файл шаблонов:
/// метод вернёт объект с функциями-шаблонами
const tpls = ss.compileFile('./myTpls.ss');

http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  
  // Вызываем шаблон foo и передаём параметры
  res.write(tpls.foo('bar', 'bla'));
  res.end();
}).listen(8888);
```

Разумется на практике мы будем использовать фреймворк для приложения, например, Express или Koa, но это не имеет значения.
Также следует отметить, что шаблоны можно транслировать предварительно (например, с помощью Gulp или Grunt) и подключать
полученные файлы, или же опять таки - можно использовать подход с WebPack.

* Генерация статических сайтов: с помощью плагинов интеграции (Gulp, Grunt, WebPack и т.д.) мы немедленно вызываем 
скомпилированный шаблон и сохраняем его результат. Следует отметить что "шаблон для вызова" либо указывается явно,
либо будет вычислен по формуле: `%fileName% || main || index || Object.keys().sort()[0];`, где `%fileName%` - имя
главного файла-шаблона без расширения (т.е. тот файл, на который мы натравливаем плагин).

* Использование транслированных в JS шаблонов на клиенте: полученные функции можно подключать через внешний тег `<script />`
или как модуль (с помощью WebPack), причём SS имеет ряд механизмов для удобной интеграции шаблонов СС с различными фреймворками,
например, Angular или Vue.

**Пример интеграции Angular + SS:**

```
- namespace myApp
- template main()
  < label
    Name:
  < input type = text | ng-model = yourName | placeholder = Enter a name here
  < hr
  < h1
    Hello {{yourName}}!
```

Snakeskin решает проблему "простыни" кода, code-reuse элементов вёрстки (через наследование, композицию, примеси и т.д.),
а Angular делает data-binding. С технической точки зрения SS генерирует шаблон, который потом использует Angular.

## Язык

Теперь, когда я описал свои мотивы и основые use-case, я расскажу про сам язык, но цель данного поста не дублирование 
[документации](http://snakeskintpl.github.io/docs/index-ru.html), а общий обзор.

### Концепция

Шаблон Snakeskin - это функция JavaScript, т.е. грубо говоря

```
- namespace myApp
- template main()
  Hello world!
```

после трансляции превратится в

```
if (exports.demo === 'undefined') {
	var myApp = exports.myApp = {};
}

exports.myApp.main = function main() {
	return 'Hello world!';
}
```

Конечно - это упрощение, но суть ясна. Шаблоны - это функции JavaScript, которые, как правило, генерируют текст 
(в виде строки, DocumentFragment или другом заданном представлении).

### Синтаксис

SS, подобно Stylus, поддерживает 2 разных вида синтаксиса:

* основанный на директивах, которые заключены в фигурные скобки, например:
 
```
{namespace myApp}
{template main(name = 'world')}
  Hello {world}!
{/template}
```

Плюс такого подхода, что его удобно использовать для генерации текста, который основан на "управляемых пробелах", например,
Markdown. Также при таком подходе "блочные" директивы (т.е. которые могут включать в себя текст) обязаны явно закрываться.

**Примечание:** для генерации текста, где используются символы фигурных скобок в SS есть [специальный механизм](http://snakeskintpl.github.io/docs/guide-ru.html#basics--%D0%A0%D0%B0%D1%81%D1%88%D0%B8%D1%80%D0%B5%D0%BD%D0%BD%D1%8B%D0%B9_%D1%81%D0%B8%D0%BD%D1%82%D0%B0%D0%BA%D1%81%D0%B8%D1%81).

* основанный на управляемых пробелах и идеологически близкий к Jade и HAML, перепишем пример данный выше:

```
- namespace myApp
- template main(name = 'world')
  Hello {world}!
```

Главные плюсы такого подхода - это краткость и наглядность, а также нет нужды в явном закрытии блочных директив.
Такой подход идеален для генерации XML подобных структур.

Также SS поддерживает смешивание синтаксисов, т.е. мы можем использовать одновременно оба подхода:

```
- namespace myApp

{template hello(name = 'world')}
  Hello {world}!
{/template}

- template main(name)
  += myApp.hello(name)
```

Подробнее про синтаксис можно узнать в [документации языка](http://snakeskintpl.github.io/docs/guide-ru.html#basics).

### code-reuse
#### Наследования

В SS каждый шаблон является классом, т.е. у него есть методы и свойства и он может наследоваться от другого шаблона.
При наследовании дочерний шаблон переопределяет родительские методы и свойства, а также может вводить новые, например:

```
- namespace myApp

/// Метод sayHello шаблона base
/// (аналог mixin в Jade)
- block base->sayHello(name = 'world')
  Hello {name}!

- template base()
  - doctype
  < html
    < head
      /// Статичный блок head
      /// (аналог блоков в Jade):
      /// чтобы сделать такой блок методом,
      /// достаточно просто добавить круглые скобки после имени
      - block head
        < title
          /// Свойство шаблона title
          {title = 'Главная страница' ?}
    
    < body
      - block body
        += self.sayHello()

/// Доопределяем родительский метод
- block child->sayHello(name = 'world')
  /// Вызов родителя
  - super
  Hello people!

/// Ввводим новый метод
- block child->go()
  Let's go!

- template child() extends myApp.base
  /// Переопределяем свойство
  - title = 'Дочерняя страница'
  
  /// Полностью переопределяем статичный блок
  - block body
    += self.go()
```

При наследовании шаблона также наследуются входные параметры, декораторы шаблона, различные модификаторы и т.д. - 
[подробнее](http://snakeskintpl.github.io/docs/guide-ru.html#inheritBasic).

#### Композиция

Т.к. все шаблоны SS функции, то любой шаблон может содержать в своём теле вызов другого шаблона и т.д.

```
- namespace myApp

- template hello(name = 'world')
  Hello {world}!

- template main(name)
  += myApp.hello(name)
```

[Подробнее](http://snakeskintpl.github.io/docs/api-ru.html#call)

#### Шаблонные литералы

SS позволяет создать переменную со значением некоторого подшаблона, передать его в другой шаблона и т.д.

```
- namespace myApp

- template wrap(content)
  < .wrapper
    {content}

- template main(name)
  += myApp.wrap()
    < .hello
      Hello world!
```

### Модули

SS может подключать другие файлы шаблоны с помощью директивы [include](http://snakeskintpl.github.io/docs/api-ru.html#include),
т.е. позволяет разбить код на логические части или создавать подключаемые библиотеки и т.д. Каждый файл образует модуль 
и глобальные переменные уровня шаблоны находятся инкапсулированы в нём, а все шаблоны - экспортируются. 

**math.ss**

```
- namespace math
- template calc(a, b)
  {a + b}
```

**app.ss**

```
- namespace myApp
- include './math'

- template main()
  += math.calc(1, 2)
```

### Основные фичи
#### Богатый набор встроенных директив

SS вводит директивы близких по семантике к JS 
([if](http://snakeskintpl.github.io/docs/api-ru.html#if), [for](http://snakeskintpl.github.io/docs/api-ru.html#for) и т.д.), 
а также ряд дополнительных ([unless](http://snakeskintpl.github.io/docs/api-ru.html#unless), [with](http://snakeskintpl.github.io/docs/api-ru.html#with) и т.д.).

SS исправляет ряд неудачных архитектурных решений JS, например, переменные SS имеют блочную область видимости (подобно let из ES2015),
а директива with устраняет недостатки удалённой из JS одноимённой конструкции и т.д.

#### Механизм фильтров

Фильтры используются в большинстве шаблонных движков, но в SS они пронизывают весь язык, т.е. мы можем использовать их
буквально везде: при создании переменных, в циклах, при декларации параметров функций и т.д.

```
- namespace myApp
- template main((str|trim), name = ('World'|lower))
  - var a = {foo: 'bar'} |json
```

[Подробнее](http://snakeskintpl.github.io/docs/guide-ru.html#filters).

#### Двусторонняя интеграция с JS

JS код может вызывать шаблоны SS, а SS может импортировать модули JS с помощью директивы [import](#http://snakeskintpl.github.io/docs/api-ru.html#import),
а также поддерживает все основные виды модулей: umd, amd, commonjs, native и global.

```
- namespace myApp
- import { readdirSync } from 'fs'

- template main((str|trim), name = ('World'|lower))
  - forEach readdirSync('./foo') => dirname
    {dirname}
```

#### Встроенный механизм локализации шаблонов

[Подробнее](http://snakeskintpl.github.io/docs/guide-ru.html#localization).

#### Встроенный механизм итеграции с другими шаблонами

[Подробнее](http://snakeskintpl.github.io/docs/guide-ru.html#introLiteral).

#### Полный контроль за пробельными символами

[Подробнее](http://snakeskintpl.github.io/docs/guide-ru.html#introTemplates--%D0%A0%D0%B0%D0%B1%D0%BE%D1%82%D0%B0_%D1%81_%D0%BF%D1%80%D0%BE%D0%B1%D0%B5%D0%BB%D1%8C%D0%BD%D1%8B%D0%BC%D0%B8_%D1%81%D0%B8%D0%BC%D0%B2%D0%BE%D0%BB%D0%B0%D0%BC%D0%B8).

Также см. раздел ["Работа с пробельными символами"](http://snakeskintpl.github.io/docs/api-ru.html#ignoreWhitespaces).

#### Поддержка ссылок на родительский класс

Подобно Sass или Stylus, в SS есть способ ссылаться на родительский класс - это удобно при использовании БЭМ подхода.

```
- namespace myApp
- template main()
  < .hello
    /// hello__wrap
    < .&__wrap
      /// hello__cont
      < .&__cont 
```

Принцип работы следующий: если при декларации тега задать имя класса, которое начинается с символа `&`, 
то он будет заменён на ближайший родительский класс, который декларировался без этого символа.

[Подробнее](http://snakeskintpl.github.io/docs/api-ru.html#tag--%D0%A1%D1%81%D1%8B%D0%BB%D0%BA%D0%B8_%D0%BD%D0%B0_%D1%80%D0%BE%D0%B4%D0%B8%D1%82%D0%B5%D0%BB%D1%8C%D1%81%D0%BA%D0%B8%D0%B9_%D0%BA%D0%BB%D0%B0%D1%81%D1%81).

#### Умная интерполяция

Многие директивы Snakeskin поддерживают механизм интерполяции, т.е. прокидывание динамических значений шаблона
в директивы, например:

```
- namespace myApp
- template main(area)
  < ${area ? 'textarea' : 'input'}.b-${area ? 'textarea' : 'input'}
    Бла бла бла
```

В данном примере SS поймёт, какой код нужно генерировать в зависимости от значений и для `area == true` это будет

```html
<textarea class="b-textarea">
  Бла бла бла
</textarea>
```

А для `area == false`

```html
<input class="b-input" value="Бла бла бла">
```

[Подробнее](http://snakeskintpl.github.io/docs/api-ru.html#tag--%D0%98%D0%BD%D1%82%D0%B5%D1%80%D0%BF%D0%BE%D0%BB%D1%8F%D1%86%D0%B8%D1%8F).

#### Интеграция со всеми основными системами сборок

[Gulp](https://github.com/SnakeskinTpl/gulp-snakeskin), [Grunt](https://github.com/SnakeskinTpl/grunt-snakeskin), [Webpack](https://github.com/SnakeskinTpl/snakeskin-loader).

#### Хорошая кодовая база

Snakeskin полностью написан на ES2015, содержит большое количество тестов и проходит максимально строгую проверку Google Closure Compiler в режиме
ADVANCED. Код хорошо документирован в соотвествии со стандартом JSDoc от Google.

#### Большая и подробная документация

Которая, кстати, написана на Snakeskin.

## Заключение

Я очень бегло рассмотрел функционал языка, но думаю, что те кто заинтересовался проследуют в [документацию](http://snakeskintpl.github.io/docs/api-ru.html)
и продолжат изучение.

Если кто желает помочь с переводом доки на английский, то прошу присылать PR на файлы в папке 
[en](https://github.com/SnakeskinTpl/docs/tree/gh-pages/docs/en).

О найденых багах пишите в [Issues](https://github.com/SnakeskinTpl/Snakeskin/issues) на GitHub-e проекта.
Задавать вопросы можно там же.

Всем спасибо за внимание и хорошего времени суток!
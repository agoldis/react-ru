---
id: lifting-state-up
title: Подъём состояния
permalink: docs/lifting-state-up.html
prev: forms.html
next: composition-vs-inheritance.html
redirect_from:
  - "docs/flux-overview.html"
  - "docs/flux-todo-list.html"
---

Часто несколько компонентов должны отражать изменения одних и тех же данных. Мы рекомендуем поднимать общее состояние до ближайшего общего предка. Давайте посмотрим, как это работает в действии.

В этом разделе мы создадим калькулятор температуры, который вычисляет, будет ли вода кипеть при определённой температуре.

Мы начнем с компонента под названием `BoilingVerdict`. Он принимает температуру по шкале Цельсия в качестве свойства и выводит, достаточно ли подходит температура для кипячения воды:

```js{3,5}
function BoilingVerdict(props) {
  if (props.celsius >= 100) {
    return <p>Вода закипит.</p>;
  }
  return <p>Вода не закипит.</p>;
}
```

Затем мы создадим компонент `Calculator`. Он отрисовывает `<input>`, позволяющий вводить температуру и сохраняет её значение в `this.state.temperature`.

Кроме того, он отрисовывает `BoilingVerdict` для отображения текущего значения, введённого в поле ввода.

```js{5,9,13,17-21}
class Calculator extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
    this.state = {temperature: ''};
  }

  handleChange(e) {
    this.setState({temperature: e.target.value});
  }

  render() {
    const temperature = this.state.temperature;
    return (
      <fieldset>
        <legend>Введите температуру в градусах Цельсия:</legend>
        <input
          value={temperature}
          onChange={this.handleChange} />
        <BoilingVerdict
          celsius={parseFloat(temperature)} />
      </fieldset>
    );
  }
}
```

[**Попробовать на CodePen**](https://codepen.io/gaearon/pen/ZXeOBm?editors=0010)

## Добавление второго поля ввода

Наше новое требование состоит в том, что в дополнение к полю ввода градусов по шкале Цельсия мы добавляем аналогичное поле ввода, но по шкале Фаренгейта; оба поля будут синхронизироваться.

Мы можем начать с извлечения компонента `TemperatureInput` из `Calculator`. Мы добавим в него новое свойство `scale`, значением которого может быть либо `"c"` или `"f"`:

```js{1-4,19,22}
const scaleNames = {
  c: 'Цельсия',
  f: 'Фаренгейта'
};

class TemperatureInput extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
    this.state = {temperature: ''};
  }

  handleChange(e) {
    this.setState({temperature: e.target.value});
  }

  render() {
    const temperature = this.state.temperature;
    const scale = this.props.scale;
    return (
      <fieldset>
        <legend>Enter temperature in {scaleNames[scale]}:</legend>
        <input value={temperature}
               onChange={this.handleChange} />
      </fieldset>
    );
  }
}
```

Теперь мы можем изменить `Calculator`, чтобы отрисовать два отдельных поля ввода температуры:

```js{5,6}
class Calculator extends React.Component {
  render() {
    return (
      <div>
        <TemperatureInput scale="c" />
        <TemperatureInput scale="f" />
      </div>
    );
  }
}
```

[**Попробовать на CodePen**](https://codepen.io/gaearon/pen/jGBryx?editors=0010)

Сейчас у нас есть два поля ввода, но когда вы вводите температуру в одно из них, другое поле не обновляется. Это противоречит нашему требованию: мы хотим их синхронизировать.

Мы также не можем отображать `BoilingVerdict` из `Calculator`. Компонент `Calculator` не знает текущую температуру, потому что он скрыт внутри `TemperatureInput`.

## Написание функций для конвертации температур

Во-первых, мы напишем две функции для конвертации градусов по шкале Цельсия в Фаренгейт и обратно:

```js
function toCelsius(fahrenheit) {
  return (fahrenheit - 32) * 5 / 9;
}

function toFahrenheit(celsius) {
  return (celsius * 9 / 5) + 32;
}
```

Эти две функции преобразуют числа. Мы напишем ещё одну функцию, которая принимает строку с температурой (`temperature`) и функцию конвертации (`convert`) в качестве аргументов, и возвращает строку. Мы будем использовать эту функцию для вычисления значения из одного поля ввода на основе значения из другого поля ввода.

Данная функция возвращает пустую строку при некорректном значении аргумента `temperature`, и округляет возвращаемое значение до трёх чисел после запятой:

```js
function tryConvert(temperature, convert) {
  const input = parseFloat(temperature);
  if (Number.isNaN(input)) {
    return '';
  }
  const output = convert(input);
  const rounded = Math.round(output * 1000) / 1000;
  return rounded.toString();
}
```

Например, вызов `tryConvert('abc', toCelsius)` возвратит пустую строку, а вызов `tryConvert('10.22', toFahrenheit)` возвращает `'50.396'`.

## Поднятие состояния

В настоящее время оба компонента `TemperatureInput` независимо хранят свои значения в локальном состоянии:

```js{5,9,13}
class TemperatureInput extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
    this.state = {temperature: ''};
  }

  handleChange(e) {
    this.setState({temperature: e.target.value});
  }

  render() {
    const temperature = this.state.temperature;
    // ...
```

Однако мы хотим, чтобы эти два поля ввода синхронизировались друг с другом. Когда мы обновляем поле ввода градусов по Цельсию, поле ввода градусов по Фаренгейту должно отражать преобразованную температуру и наоборот.

В React общее состояние (состояние, разделяемое между компонентами) достигается путём перемещения компонента до ближайшего общего предка, в котором тот находится. Это называется «подъём состояния». Мы удалим локальное состояние из `TemperatureInput` и переместим его в `Calculator`.

Если `Calculator` владеет общим состоянием, он становится «источником истины» текущей температуры в обоих полей ввода. Он может предоставить им двоим значения, которые не противоречат друг другу. Поскольку свойства обоих компонентов `TemperatureInput` приходят из одного и того же родительского компонента `Calculator`, два поля ввода всегда будут синхронизироваться.

Давайте посмотрим, как это работает шаг за шагом.

Во-первых, мы заменим `this.state.temperature` на `this.props.temperature` в компоненте `TemperatureInput`. Пока давайте представим, что `this.props.temperature` уже существует, хотя нам нужно будет передать его из `Calculator` в будущем:

```js{3}
  render() {
    // Ранее было так: const temperature = this.state.temperature;
    const temperature = this.props.temperature;
    // ...
```

Мы знаем, что [свойства доступны только для чтения](/docs/components-and-props.html#props-are-read-only). Когда свойство `temperature` находилась в локальном состоянии, `TemperatureInput` мог просто вызвать `this.setState()` для изменения его значения. Однако теперь, когда `temperature` находится в родительском компоненте в качестве свойства, `TemperatureInput` не может контролировать его.

В React это обычно решается путём создания «контролируемого» компонента. Точно так же, как DOM-элемент `<input>` принимает значения `value` и `onChange`, то и пользовательский `TemperatureInput` принимает оба свойства `temperature` и `onTemperatureChange` из своего родительского` Calculator`.

Теперь, когда `TemperatureInput` хочет обновить свою температуру, он вызывает `this.props.onTemperatureChange`:

```js{3}
  handleChange(e) {
    // Ранее было так: this.setState({temperature: e.target.value});
    this.props.onTemperatureChange(e.target.value);
    // ...
```

> Примечание:
>
> В пользовательских компонентах нет особого смысла в именах свойств `temperature` или `onTemperatureChange`. Мы могли бы назвать их как-то иначе, например, `value` и` onChange`, т.к. подобные имена — распространённое соглашение.

Свойство `onTemperatureChange` будет предоставлено вместе со свойством `temperature` родительским компонентом `Calculator`. Он будет обрабатывать изменение путём изменения собственного локального состояния, тем самым повторно отображая оба поля ввода с новыми значениями. Мы вскоре рассмотрим новую реализацию `Calculator`.

Прежде чем погрузиться в изменения `Calculator`, давайте вспомним сделанные изменения в компонент `TemperatureInput`. Мы удалили из него локальное состояние, и вместо того использования `this.state.temperature` теперь используем `this.props.temperature` для получения значения температуры. Вместо вызова `this.setState()`, когда мы хотим внести изменения, теперь вызываем `this.props.onTemperatureChange()`, который передаётся компонентом `Calculator`:

```js{8,12}
class TemperatureInput extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
  }

  handleChange(e) {
    this.props.onTemperatureChange(e.target.value);
  }

  render() {
    const temperature = this.props.temperature;
    const scale = this.props.scale;
    return (
      <fieldset>
        <legend>Введите градусы по шкале {scaleNames[scale]}:</legend>
        <input value={temperature}
               onChange={this.handleChange} />
      </fieldset>
    );
  }
}
```

Теперь перейдём к компоненту `Calculator`.

Мы будем хранить текущие значения свойств `temperature` и `scale` в локальном состоянии. Это состояние, которое мы «подняли» от полей ввода, и теперь оно будет служить «источником истины» для них обоих. Это минимальное представление всех данных, про которое нам нужно знать для отрисовки обоих полей ввода.

Например, если мы вводим 37 как значение поля ввода для температуры по шкале Цельсия, состояние компонента `Calculator` будет:

```js
{
  temperature: '37',
  scale: 'c'
}
```

Если позднее мы изменим поле для ввода градусов по шкале Фаренгейта на 212, состояние `Calculator` будет:

```js
{
  temperature: '212',
  scale: 'f'
}
```

Мы могли бы сохранить значение обоих полей ввода, но это оказалось бы ненужным. Достаточно сохранить значение последнего изменённого поля ввода и шкалу, которая это значение представляет. Затем мы можем вывести значение для другого поля ввода, основываясь только на текущих значениях `temperature` и `scale`.

Поля вводов остаются в синхронизации, поскольку их значения вычисляются из одного и того же состояния:

```js{6,10,14,18-21,27-28,31-32,34}
class Calculator extends React.Component {
  constructor(props) {
    super(props);
    this.handleCelsiusChange = this.handleCelsiusChange.bind(this);
    this.handleFahrenheitChange = this.handleFahrenheitChange.bind(this);
    this.state = {temperature: '', scale: 'c'};
  }

  handleCelsiusChange(temperature) {
    this.setState({scale: 'c', temperature});
  }

  handleFahrenheitChange(temperature) {
    this.setState({scale: 'f', temperature});
  }

  render() {
    const scale = this.state.scale;
    const temperature = this.state.temperature;
    const celsius = scale === 'f' ? tryConvert(temperature, toCelsius) : temperature;
    const fahrenheit = scale === 'c' ? tryConvert(temperature, toFahrenheit) : temperature;

    return (
      <div>
        <TemperatureInput
          scale="c"
          temperature={celsius}
          onTemperatureChange={this.handleCelsiusChange} />
        <TemperatureInput
          scale="f"
          temperature={fahrenheit}
          onTemperatureChange={this.handleFahrenheitChange} />
        <BoilingVerdict
          celsius={parseFloat(celsius)} />
      </div>
    );
  }
}
```

[**Попробовать на CodePen**](https://codepen.io/gaearon/pen/WZpxpz?editors=0010)

Теперь, независимо от того, какое поля ввода вы редактируете, `this.state.temperature` и `this.state.scale` в `Calculator` обновляются. Одно из полей ввода получает значение как есть, поэтому любое пользовательское поле ввода сохраняется, а значение другого поля ввода всегда пересчитывается на его основе.

Давайте посмотрим, что происходит, когда вы редактируете поле ввода:

* React вызывает функцию, указанную в `onChange` на DOM-элементе `<input>`. В нашем случае это метод `handleChange()` компонента `TemperatureInput`.
* Метод `handleChange()` в компоненте` TemperatureInput` вызывает `this.props.onTemperatureChange()` с новым требуемым значением. Его свойства, включая `onTemperatureChange`, были предоставлены его родительским компонентом — `Calculator`.
* Когда он был ранее отрисован, `Calculator` указывает, что `onTemperatureChange` в компоненте `TemperatureInput` по шкале Цельсия — это метод `handleCelsiusChange` в компоненте `Calculator`, а `onTemperatureChange` компонента `TemperatureInput` по шкале Фаренгейта — это метод `handleFahrenheitChange` в компоненте `Calculator`. Поэтому любой из этих двух методов `Calculator` вызывается в зависимости от того, какое поле ввода отредактировано.
* Внутри этих методов компонент `Calculator` указывает React сделать повторную отрисовку при вызове `this.setState()` со значением нового поля ввода и текущей шкалой.
* React вызывает метод `render()` компонента` Calculator`, чтобы узнать, как должен выглядеть пользовательский интерфейс. Значения обоих полей ввода пересчитываются исходя из текущей температуры и шкалы. В этом методе выполняется конвертация температуры.
* React вызывает методы `render()` конкретных компонентов `TemperatureInput` с их новыми свойствами, переданные компонентом `Calculator`. Он узнает, как должен выглядеть пользовательский интерфейс.
* React вызывает метод `render()` компонента `Boiling Verdict`, передавая температуру в градусах Цельсия в качестве свойства.
* DOM React обновляет DOM, чтобы привести его в соответствие с введёнными значениями в полях ввода. Поле ввода, которое было только что отредактировано, отражает его текущее значение, а другое поле ввода обновляется до температуры после конвертации.

Каждое обновление проходит через одни и те же шаги, поэтому поля ввода всегда синхронизируются.

## Извлечённые уроки

Для любых изменяемых данных в React-приложении должен быть один «источник истины». Обычно состояние сначала добавляется к компоненту, которому оно требуется для отрисовки. Затем, если другие компоненты также нуждаются в нём, вы можете поднять его до ближайшего общего предка. Вместо того, чтобы пытаться синхронизировать состояние между различными компонентами, вы должны полагаться на [нисходящий поток данных (поток данных сверху вниз)](/docs/state-and-lifecycle.html#the-data-flows-down).

Состояние, которое поднимается, включает в себя написание больше «шаблонного» кода, чем подходы с двусторонней привязкой данных, но как из преимуществ получаем меньше затрат для поиска и исправления багов. Так как любое состояние «живёт» в каком-нибудь компоненте, и только этот компонент может его изменить, возможность совершить баги значительно уменьшается. Кроме того, вы можете реализовать любую пользовательскую логику для отклонения или преобразования пользовательского значения поля ввода.

Если что-то может быть получено либо из свойств, либо из состояния, оно, вероятно, не должно находиться в состоянии. Например, вместо сохранения `celsiusValue` и `fahrenheitValue`, мы сохраняем только последнюю введённую температуру (`temperature`) и её шкалу (`scale`). Значение другого поля ввода всегда можно вычислить, ориентируясь на эти значения, в методе `render()`. Это позволяет очистить или применить округление значение другого полю, не теряя при этом точности пользовательских данных в поле ввода.

Когда вы видите, что в пользовательском интерфейсе что-то не так, вы можете использовать [React Developer Tools](https://github.com/facebook/react-devtools) для проверки свойств и навигации по дереву компонентов до тех пор, пока не найдёте тот компонент, который ответственный за обновление состояния. Это позволяет отследить источник багов:

<img src="../images/docs/react-devtools-state.gif" alt="Мониторинг состояния в React DevTools" max-width="100%" height="100%">

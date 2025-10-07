<language-switcher/>
# Начало

## Основы ECS
Becsy это Entity Component System (ECS) фреймворк для веб-приложений. Основная идея этого паттерна проектирования в том, чтобы отказаться от представления сущностей приложения через иерархию классов в пользу композиции и парадигм программирования, ориентированного на данные. ([Подробнее на википедии](https://en.wikipedia.org/wiki/Entity_component_system)). Организация приложений на ECS архитектуре может сделать код более эффективным и лёгким для расширения.

Основные понятия ECS включают:
- [сущности](ru/architecture/entities): объект с уникальным идентификатором, имеющий множество компонентов.
- [компоненты](ru/architecture/components): Различные характеристики сущностей, такие как форма, физический свойства, хитбоксы. Компонент это единственное место, где хранятся данные сущностей.
- [системы](ru/architecture/systems): часть кода, которая содержит и выполняет логику приложения, путём чтения и изменения компонентов сущностей.
- [запросы](ru/architecture/queries): используются системами, чтобы на основе компонентов определить, какие сущности они обрабатывают.
- [мир](ru/architecture/world): контейнер для сущностей, компонентов, систем и запросов.

Обычное ECS-приложение можно охарактеризовать так:
1. Определение типов *компонентов* на основе данных, обрабатываемых приложением.
2. Создание *сущностей* и добавление в них *компонентов*.
3. Описание *систем*, которые будут использовать *компоненты* чтобы получить и преобразовать данные *сущностей*, полученных *запросом*.
4. Выполнение всех *систем* в каждом кадре.

## Добавление Becsy в проект
Becsy опубликован на `npm` как `@lastolivegames/becsy`.
```bash
npm install @lastolivegames/becsy
```

## Создание мира
Мир это контейнер для сущностей, компонентов и систем. Becsy поддерживает только один одновременно работающий мир.

Давайте создадим наш первый мир:
```ts
const world = await World.create();
```
```js
const world = await World.create();
```

## Объявление компонентов
Компоненты это просто объекты, которые хранят данные. Мы объявляем их как классы без логики, содержащие дополнительные метаданные о своих свойствах.

```js
class Acceleration {
  static schema = {
    value: {type: Type.float64, default: 0.1}
  };
}

class Position {
  static schema = {
    x: {type: Type.float64},
    y: {type: Type.float64},
    z: {type: Type.float64}
  };
}
```
```ts
@component class Acceleration {
  @field({type: Type.float64, default: 0.1}) declare value: number;
}

@component class Position {
  @field.float64 declare x: number;
  @field.float64 declare y: number;
  @field.float64 declare z: number;
}
```

::: only-ts
Декоратор `@component` автоматически зарегистрирует тип компонента в мире (не забудьте включить `"experimentalDecorators": true` в `tsconfig.json`).
:::

::: only-js
Нам нужно дать миру знать о наших компонентах, передавая их при создании мира:

```js
const world = await World.create({defs: [Acceleration, Position]});
```
:::

[Подробнее о том, как определять типы компонентов](ru/architecture/components).

## Создание сущностей
Теперь, когда у нас есть мир и компоненты, пора создать [сущности](architecture/entities) и добавить в них экземпляры компонентов:
```js
world.createEntity(Position);
for (let i = 0; i < 10; i++) {
  world.createEntity(
    Acceleration,
    Position, {x: Math.random() * 10, y: Math.random() * 10, z: 0}
  );
}
```
```ts
world.createEntity(Position);
for (let i = 0; i < 10; i++) {
  world.createEntity(
    Acceleration,
    Position, {x: Math.random() * 10, y: Math.random() * 10, z: 0}
  );
}
```

Только что мы создали 11 сущностей: 10 с компонентами `Acceleration` и `Position`, и одну с компонентом `Position`. Обратите внимание, что мы добавляем компонент `Position` с параметрами. Без них компонент будет использовать значения по-умолчанию, указанные при объявлении свойств, или глобальные значения по-умолчанию (0 для чисел, false для bool и т.д.).

[Подробнее о том, как создавать и хранить сущности](ru/architecture/entities).

## Создание систем
Теперь пришло время объявить [системы](eu/architecture/systems), которые будут обрабатывать только что созданные компоненты. Система должна наследоваться от класса `System` и может переопределять методы жизненного цикла системы, как мы переопределили метод `execute`, который вызывается в каждом кадре, ниже. Кроме того, нам нужно объявить [запросы](ru/architecture/queries), чтобы получить сущности, содержащие компоненты, которые мы планируем обрабатывать.

Для начала создадим систему, которая будет проходить по всем сущностям, содержащим компонент `Position` (мы создали 11 таких ранее) и выводить в консоль их положение.

```js
class PositionLogSystem extends System {
  // Запрос к сущностям, которые содержат компонент "Position".
  entities = this.query(q => q.current.with(Position));

  // Этот метод будет вызван в каждом кадре.
  execute() {
    // Цикл по всем сущностям в запросе.
    for (const entity of this.entities.current) {
      // Получить компонент `Position` из текущей сущности.
      const pos = entity.read(Position);
      console.log(
        `Сущность с номером ${entity.ordinal} содержит компонент ` +
        `Position={x: ${pos.x}, y: ${pos.y}, z: ${pos.z}}`
      );
    }
  }
}
```
```ts
@system class PositionLogSystem extends System {
  // Запрос к сущностям, которые содержат компонент "Position".
  entities = this.query(q => q.current.with(Position));

  // Этот метод будет вызван в каждом кадре.
  execute() {
  // Цикл по всем сущностям в запросе.
    for (const entity of this.entities.current) {
    // Получить компонент `Position` из текущей сущности.
      const pos = entity.read(Position);
      console.log(
        `Сущность с номером ${entity.ordinal} содержит компонент ` +
        `Position={x: ${pos.x}, y: ${pos.y}, z: ${pos.z}}`
      );
    }
  }
}
```

Следующая система перемещает каждую сущность, у которой есть компоненты `Position` и `Acceleration`.

```js
class MovableSystem extends System {
  // Запрос за сущностями, у которых есть компоненты "Acceleration" и "Position",
  // с уточненением что нам нужен доступ на чтение компонента "Acceleration", а также
  // на чтение и запись компонента "Position".
  entities = this.query(
    q => q.current.with(Acceleration).read.and.with(Position).write);

  // Этот метод будет вызван в каждом кадре.
  execute() {
    // Цикл по всем сущностям в запросе.
    for (const entity of this.entities.current) {
      // Получить компонент `Acceleration` с доступом только на чтение и извечь из него значение.
      const acceleration = entity.read(Acceleration).value;

      // Получить компонент `Position` для чтения и записи.
      const position = entity.write(Position);
      position.x += acceleration * this.delta;
      position.y += acceleration * this.delta;
      position.z += acceleration * this.delta;
    }
  }
}
```
```ts
@system class MovableSystem extends System {
  // Запрос за сущностями, у которых есть компоненты "Acceleration" и "Position",
  // с уточненением что нам нужен доступ на чтение компонента "Acceleration", а также
  // на чтение и запись компонента "Position".
  entities = this.query(
    q => q.current.with(Acceleration).read.and.with(Position).write);

  // Этот метод будет вызван в каждом кадре.
  execute() {
    // Цикл по всем сущностям в запросе.
    for (const entity of this.entities.current) {
      // Получить компонент `Acceleration` с доступом только на чтение и извечь из него значение.
      const acceleration = entity.read(Acceleration).value;

      // Получить компонент `Position` для чтения и записи.
      const position = entity.write(Position);
      position.x += acceleration * this.delta;
      position.y += acceleration * this.delta;
      position.z += acceleration * this.delta;
    }
  }
}
```

Запрос в этой системе содержит сущности, у которых есть и `Acceleration` и `Position` - 10 в нашем примере.

Обратите внимание, как мы получаем доступ к компонентам сущности:
- `read(Component)`: если нам нужно только прочитать данные компонента.
- `write(Component)`: если нам нужна возможность записи данных в компонент.

И что объявляя запрос, мы должны указать какие доступы к каким компонентам нам нужны, иначе работа системы завершится с ошибкой.

При необходимости, одна система может содержать несколько запросов и работать с каждым из них в методе `execute`, например:
```js
class SystemDemo extends System {
  boxes = this.query(q => q.current.with(Box));
  balls = this.query(q => q.current.with(Ball));

  execute() {
    for (const entity of this.boxes.current) { /* обработка коробок (box) */ }
    for (const entity of this.balls.current) { /* обработка мячей (ball) */ }
  }
}
```
```ts
@system class SystemDemo extends System {
  boxes = this.query(q => q.current.with(Box));
  balls = this.query(q => q.current.with(Ball));

  execute() {
    for (const entity of this.boxes.current) { /* обработка коробок (box) */ }
    for (const entity of this.balls.current) { /* обработка мячей (ball) */ }
  }
}
```

::: only-js
Системы нужно регистрировать в создаваемом мире, прямо как компоненты:

```js
const world = await World.create({
  defs: [Acceleration, Position, PositionLogSystem, MovableSystem]
});
```
:::

Подробнее о [системах](ru/architecture/systems) и [запросах](ru/architecture/queries).

## Запуск систем
Теперь остаётся только вызывать `world.execute()` в каждом кадре. Becsy не имеет встроенного функционала для этого, так что вы можете сделать что-то вроде:
```js
async function run() {
  // Выполнить все системы
  await world.execute();
  requestAnimationFrame(run);
}

run();
```
```ts
async function run() {
  // Выполнить все системы
  await world.execute();
  requestAnimationFrame(run);
}

run();
```

## Итоговый код примера
```js
import {System, Type, World} from '@lastolivegames/becsy';

class Acceleration {
  static schema = {
    value: {type: Type.float64, default: 0.1}
  };
}

class Position {
  static schema = {
    x: {type: Type.float64},
    y: {type: Type.float64},
    z: {type: Type.float64}
  };
}

class PositionLogSystem extends System {
  entities = this.query(q => q.current.with(Position));

  execute() {
    for (const entity of this.entities.current) {
      const pos = entity.read(Position);
      console.log(
        `Сущность с номером ${entity.ordinal} содержит компонент ` +
        `Position={x: ${pos.x}, y: ${pos.y}, z: ${pos.z}}`
      );
    }
  }
}

class MovableSystem extends System {
  entities = this.query(
    q => q.current.with(Acceleration).read.and.with(Position).write);

  execute() {
    for (const entity of this.entities.current) {
      const acceleration = entity.read(Acceleration).value;
      const position = entity.write(Position);
      position.x += acceleration * this.delta;
      position.y += acceleration * this.delta;
      position.z += acceleration * this.delta;
    }
  }
}

const world = await World.create({
  defs: [Acceleration, Position, PositionLogSystem, MovableSystem]
});


world.createEntity(Position);
for (let i = 0; i < 10; i++) {
  world.createEntity(
    Acceleration,
    Position, {x: Math.random() * 10, y: Math.random() * 10, z: 0}
  );
}

async function run() {
  await world.execute();
  requestAnimationFrame(run);
}

run();
```
```ts
import {component, field, system, System, Type, World} from '@lastolivegames/becsy';

@component class Acceleration {
  @field({type: Type.float64, default: 0.1}) declare value: number;
}

@component class Position {
  @field.float64 declare x: number;
  @field.float64 declare y: number;
  @field.float64 declare z: number;
}

@system class PositionLogSystem extends System {
  entities = this.query(q => q.current.with(Position));

  execute() {
    for (const entity of this.entities.current) {
      const pos = entity.read(Position);
      console.log(
        `Сущность с номером ${entity.ordinal} содержит компонент ` +
        `Position={x: ${pos.x}, y: ${pos.y}, z: ${pos.z}}`
      );
    }
  }
}

@system class MovableSystem extends System {
  entities = this.query(
    q => q.current.with(Acceleration).read.and.with(Position).write);

  execute() {
    for (const entity of this.entities.current) {
      const acceleration = entity.read(Acceleration).value;
      const position = entity.write(Position);
      position.x += acceleration * this.delta;
      position.y += acceleration * this.delta;
      position.z += acceleration * this.delta;
    }
  }
}

const world = await World.create();

world.createEntity(Position);
for (let i = 0; i < 10; i++) {
  world.createEntity(
    Acceleration,
    Position, {x: Math.random() * 10, y: Math.random() * 10, z: 0}
  );
}

async function run() {
  await world.execute();
  requestAnimationFrame(run);
}

run();
```


## Что дальше?
Это был простой пример того, как вещи устроены в Becsy, для более детального ознакомления рекомендуем прочитать об [архитектуре](architecture/overview). Также вы можете ознакомиться [с другими примерами](./examples/overview) или присоединиться к нашему [каналу в Discord](https://discord.gg/X72ct6hZSr)!

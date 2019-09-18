# Посилання

Для того щоб не клонувати моделі і побороти заборону MST на те, щоб у дереві був тільки один екземпляр тієї ж моделі,використаємо концепцію посилань — [Reference](https://github.com/mobxjs/mobx-state-tree#references-and-identifiers). В цьому плані mobx-state-tree — ідеально підходить.

Для цього спершу нам потрібно вказати поле `identifier` для моделі, на яку будемо посилатись, в нашому випадку це буде модель Todo:

> Важливо: може існувати тільки один екземпляр моделі з таким ж значенням `identifier`.

```js
const TodoModel = types.model({
  // замість types.string, використаємо types.identifier
  // в більшій мірі вони ідентичні — перевірка відбувається на string.
  // якщо потрібно використовувати number, для цього є identifierNumber.
  // Відмінність identifier в тому, що він чітко вказує MST
  // яке з полів є ідентифікатором моделі
  id: types.identifier,
  //...решту полів
})
.actions((self) => ({
  // ...решту екшенів
}));
```

Коли ми вказали яке з полів буде ідентифікатором моделі, можемо посилатись на неї використовуючи `types.reference`.

```js
const GroupModel = types.model({
  id: types.string,
  title: types.string,
  // просто вказуємо, що кожен елемент масиву — посилання на модель Todo
  todos: types.array(types.reference(TodoModel)),
})
.actions((self) => ({
  addTodo(todo) {
    // а тут todo може бути як екземпляром моделі
    // так і просто значенням поля identifier
    self.todos.unshift(todo);
  },
}));
```

Отож, давайте запустимо наш код:

```js
const rootStore = RootModel.create({});

rootStore.todos.add('Яйця');
const todo = rootStore.todos.list[0];

rootStore.groups.add('Покупки');
const group = rootStore.groups.list[0];

group.addTodo(todo);
// або добавляємо, використовуючи поле, позначене як identifier
group.addTodo(todo.id);

const todoInGroup = rootStore.groups.list[0].todos[0];
todoInGroup.toggleCompleted();

const todoInList = rootStore.todos.list[0];
todoInList.toggleFavorite();

prettyPrint(rootStore);
```

І у вас повинно получитись щось типу:

```json
{
  "todos": {
    "list": [
      {
        "id": "f564c334-84a5-4230-9a65-5ce25ef36d93",
        "title": "Яйця",
        "isCompleted": true,
        "isFavorite": true
      }
    ]
  },
  "groups": {
    "list": [
      {
        "id": "b35bef5c-efa8-48c2-8751-ae8c5bea302a",
        "title": "Покупки",
        "todos": [
          "f564c334-84a5-4230-9a65-5ce25ef36d93",
          "f564c334-84a5-4230-9a65-5ce25ef36d93"
        ]
      }
    ]
  }
}
```

> Зверніть увагу на те, що `group[0].todos` в снепшоті представлені як масив `identifier` полів наших моделей (в даному випадку ми додали ту ж модель двічі) — масив посилань.

Тепер коли ми звертаємось до якоїсь з моделей, які зберігаються в масиві `todos` конкретної групи, насправді ж ми звертаємось до екземпляру моделі, яка зберігається в `todos.list`. Це можна легко перевірити, порівнявши об'єкти:

```js
console.log(todoInList === todoInGroup) // true
```
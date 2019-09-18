# Root—модель

Після тривалого використання нашого застосунку, ми виявили, що він ідеально підходить не тільки для покупок, а ще для списку справ, про котрі потрібно не забути, тому було вирішено добавити підтримку груп.

Спершу розробимо модель групи:

```js
const GroupModel = types.model({
  id: types.string,
  title: types.string,
  todos: types.array(TodoModel),
})
.actions((self) => ({
  // для того щоб мати змогу додавати в групу тудушку
  addTodo(todo) {
    self.todos.unshift(todo);
  },
}));
```

Тепер було б добре мати модель, де ми будемо зберігати всі наші групи:

```js
const GroupListModel = types.model({
  list: types.array(GroupModel),
})
.actions((self) => ({
  // для того щоб мати змогу додавати групу
  add(title) {
    const group = {
      id: uuid(),
      title,
    };

    self.list.unshift(group);
  },
}));
```

А для того щоб це все зібрати до купи, при використанні mobx-state-tree прийнято робити модель, яка буде групувати всі інші моделі і буде добавимо RootModel, яка буде мати в собі як TodoListModel, так і GroupListModel:

```js
const RootModel = types.model({
  // тут optional використовується для того, щоб моделі автоматично
  // ініціалізувались із пустим об'єктом як снепшот при ініціалізації
  // моделі RootModel
  // так як і в цьому випадку MST за нас робить під капотом `TodoListModel.create`
  todos: types.optional(TodoListModel, {}),
  groups: types.optional(GroupListModel, {}),
})
```

Давайте спробуємо це все запустити:

```js
// ініціалізуємо наш рут-стор
const rootStore = RootModel.create({});

// логуємо все
autorun(() => prettyPrint(rootStore));

// додаємо тудушку
rootStore.todos.add('Яйця');
const todo = rootStore.todos.list[0];

// створюємо групу
rootStore.groups.add('Покупки');
const group = rootStore.groups.list[0];

// додаємо тудушку в групу
group.addTodo(todo);
```

І бачимо помилку:
> Cannot add an object to a state tree if it is already part of the same or another state tree. Tried to assign an object to '/groups/list/0/todos/0', but it lives already at '/todos/list/0'

MST не дає нам додавати двічі ту ж тудушку в дерево, кожна модель має бути представлена єдиним екземпляром.

Проте проблему можна вирішити, використавши утилітку, яку надає нам mobx-state-tree – `clone`:

```js
// клонуємо тудушку.
// ця операція створює лише новий екземпляр моделі в дереві — ноду
// тобто контент може бути ідентичний, але зараз все працює
const todoClone = clone(todo);
// додаємо клоновану тудушку в групу
group.addTodo(todoClone);
```

Останнім результатом `autorun` повинен бути json, схожий на цей:

```json
{
  "todos": {
    "list": [
      {
        "id": "7db4a95d-60fd-4db8-a2d6-d1c1b07af818",
        "title": "Яйця",
        "isCompleted": false,
        "isFavorite": false
      }
    ]
  },
  "groups": {
    "list": [
      {
        "id": "be80e8de-7dda-491d-8449-3691007bcd8c",
        "title": "Покупки",
        "todos": [
          {
            "id": "7db4a95d-60fd-4db8-a2d6-d1c1b07af818",
            "title": "Яйця",
            "isCompleted": false,
            "isFavorite": false
          }
        ]
      }
    ]
  }
}
```

А тепер давайте спробуємо відмітити наші тудушку як виконану в загальному списку `todos.list[0]` та добавимо в обрану ту, що в `groups.list[0].todos[0]`:

```js
// ...решту дій з попереднього прокладу

const todoInGroup = rootStore.groups.list[0].todos[0];
todoInGroup.toggleCompleted();

const todoInList = rootStore.todos.list[0];
todoInList.toggleFavorite();

prettyPrint(rootStore);
```

І бачимо таку картину:

```json
{
  "todos": {
    "list": [
      {
        "id": "a8f549bf-c73a-44b5-9209-43a9760f3d81",
        "title": "Яйця",
        "isCompleted": false,
        "isFavorite": true
      }
    ]
  },
  "groups": {
    "list": [
      {
        "id": "1a0d0cd3-261a-4639-9187-c38de2a9273d",
        "title": "Покупки",
        "todos": [
          {
            "id": "a8f549bf-c73a-44b5-9209-43a9760f3d81",
            "title": "Яйця",
            "isCompleted": true,
            "isFavorite": false
          }
        ]
      }
    ]
  }
}
```

В нас є проблема: коли ми змінюємо нашу тудушку з списку всіх тудушок, то ця що в списку тудушок групи "Покупки" не змінюється, і навпаки — коли ми змінюємо тудушку в групі, тудушка в загальному списку залишається без змін, а це явно не те, чого б ми хотіли, так як в них id – ідентичні. З нашої точки зору це одна й та ж тудушка, проте для MST – це зовсім різні моделі, так як ми зробили її клон за допомогою `clone(todo)`.

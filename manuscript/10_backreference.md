# Зворотнє посилання

Ми додали можливість створення нової тудушки в загальному списку. Проте що, якщо ми хочемо її додати відразу до якоїсь групи? Можна спробувати наступним чином:

1. Спочатку нам треба знати `id` тудушки в групі, для того наш екшен `add` повинен повертати тудушку, яку створив:

```js
const TodoListModel = types.model({
  list: types.array(TodoModel),
})
.actions((self) => ({
  add(title) {
    // створюємо новий об'єкт тудушки
    const todo = {
      id: uuid(),
      title,
    };

    // додаємо в масив
    self.list.unshift(todo);

    // і повертаємо там, де нам буде потрібно
    return todo;
  }
}));
```

2. Далі нам потрібно добавити метод додавання тудушки до нашої групи, проте з поправкою на те, що нам потрібно створити тудушку:

```js
const GroupModel = types.model({
  // всі поля
  // а тут заміняємо `reference` на `safeReference`
  // тому що в момент заміни моделі з таким ідентифікатором може ще не існувати
  todos: types.array(types.safeReference(TodoModel)),
})
.actions((self) => ({
  // ...всі екшени
  createAndAddTodo(text) {
    // MST надає нам можливість отримати посилання на RootStore будь де
    const todo = getRoot(self).todos.add(title);

    // використовуємо екшен, який ми написали до того
    self.addTodo(todo.id);
  },
}));
```

Проте після запуску ми виявляємо проблему: коли ми створюємо і додаємо нову тудушку — насправді ми працюємо з тимчасовим ідентифікатором, який буде замінений на справжній, тобто той, який створить для нас сервер. Отже через деякий час посилання на TodoModel, яке ми передали через `self.addTodo(todo.id)` буде вже невалідне.

Так як тудушка в нашому випадку може належати тільки до одної групи, найпростішим рішенням цієї проблеми буде використання зворотнього посилання на групу в тудушці — тобто група буде мати багато тудушок, а кожна тудушка буде мати одну групу.

Для цього ми спершу оголосимо опціональне посилання в моделі Todo на модель Group:

```js
const TodoModel = types.model({
  // ...всі поля моделі

  // maybe тут означає, що це поле може бути `undefined`
  group: t.maybe(t.reference(GroupModel)),
})
.actions((self) => ({
  // ...решту екшенів
}));
```

Коли ми створюємо нашу тудушку, ми також прокидаємо в об'єкт `todo` ідентифікатор нашої групи:

```js
const TodoListModel = types.model({
  list: types.array(TodoModel),
})
.actions((self) => ({
  // додаємо тут необов'язковий параметр `group`
  add(title, group) {
    // створюємо новий об'єкт тудушки
    const todo = {
      id: uuid(),
      title,
      // не забуваємо його додати до снепшоту
      group,
    };

    // додаємо в масив
    self.list.unshift(todo);
  }
}));

const GroupModel = types.model({ /* поля моделі */ })
.actions((self) => ({
  // ...всі екшени
  createAndAddTodo(text) {
    // зверніть увагу що ми передаємо `id` моделі
    const todo = getRoot(self).todos.add(title, self.id);

    // додаємо в список всіх тудушок групи
    self.addTodo(todo.id);
  },

  // а цей екшен буде використаний чуть нижче
  replaceTodo(id, todo) {
    // шукаємо потрібний індекс
    const index = self.todos.findIndex(item => item.id === id);

    if (index > -1) {
      // заміняємо елемент за цим індексом на новий
      self.todos[index] = todo;
    }
  }
}));
```

А в самій моделі Todo дещо модифікуємо наш `send` екшен:

```js
const TodoModel = types.model({
 // ...решту полів моделі
})
.actions((self) => ({
  // ...решту екшенів

  send: flow(function* sendFlow() {
    self.isErrorSending = false;
    self.isSending = true;

    // зберігаємо попередній ідентифікатор
    const oldId = self.id;

    try {
      const todo = yield Api.Todos.create(self);
      // оновляєм, мутуючи нашу модель
      // в тому числі заміняємо id на новий
      Object.assign(self, json);

      self.isSending = false;

      // перевіряємо чи додали тудушку через групу
      if (self.group) {
        // заміняємо попередній елемент в списку todos нашої групи на новий
        self.group.replaceTodo(oldId, self);
      }
    } catch(err) {
      self.isSending = false;
      self.isErrorSending = true;
    }
  }),
}));
```

Таким чином ми заміняємо ідентифікатор, використовуючи зворотнє посилання з нашої тудушки на групу, в якій ми її створили.

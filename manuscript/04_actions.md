# Екшени

Відображати дані — круто, але було б добре ще їх додавати. В даному випадку нам потрібно додати новий елемент в масив `list`. Давайте спробуємо:

```js
// створимо новий екземпляр нашої задачі
const todo = TodoModel.create({ id: uuid(), title: 'Яйці' });
// додаємо в масив, просто мутуючи його
todoList.list.push(todo)
```

Але отримуємо помилку:
> Cannot modify 'AnonymousModel[]@/list', the object is protected and can only be modified by using an action.

А все тому що для того щоб змінити дані нашої моделі, нам потрібно використати [екшен](https://github.com/mobxjs/mobx-state-tree#actions). MST не дозволяє нам змінювати нашу модель поза екшеном. Отож, давайте добавимо наш екшен:

```js
const TodoListModel = types.model({
  list: types.array(TodoModel),
})
.actions((self) => ({
  add(title) {
    // створюємо новий об'єкт задачі, додаючи всі поля
    // які не позначені як optional — так званий снепшот моделі
    const todo = {
      id: uuid(),
      title,
    };

    // для того щоб додати задачу в масив із TodoModel
    // нам потрібно було б її створити
    // const todoNode = TodoModel.create(todo);
    // self.list.unshift(todoNode);
    // проте за нас це робить MST, тому ми можемо прокинути тільки сам снепшот
    // і так: ми мутуємо наш масив
    self.list.unshift(todo);
  }
}));
```

Мати змогу додавати елементи — добре, проте хотілось би ще відмічати що вже куплено, а що ще ні, для того добавимо поле `isCompleted` та екшен для зміни до нашої моделі:

```js
const TodoModel = types.model({
  id: types.string,
  title: types.string,
  isCompleted: types.optional(types.boolean, false),
})
.actions((self) => ({
  toggleCompleted() {
    // і знову ж таки ми мутуємо нашу модель, так-так
    self.isCompleted = !self.isCompleted;
  },
}));
```

Давайте створимо пару задач та змінимо їх `isCompleted` поля, попутно логуючи всі зміни за допомогою функції [autorun](https://mobx.js.org/refguide/autorun.html), яку нам надає `mobx`

```js
const todoList = TodoListModel.create({});
// просто красиво виводимо снепшот нашої моделі
// autorun буде викликано кожного разу, як щось змінюється в todoList
// (трошки магічно, знаю, але це ми розберемо іншого разу)
autorun(() => prettyPrint(todoList));

todoList.add('Яйця');
const todo = todoList.list[0];

todo.toggleCompleted();
```

Останнім результатом виклику колбеку autorun повинен бути цей json:

```json
{
  "list": [
    {
      "id": "14885c7b-a1b3-4ddb-94d2-0e3fd689fe24",
      "title": "Яйця",
      "isCompleted": true
    }
  ]
}
```

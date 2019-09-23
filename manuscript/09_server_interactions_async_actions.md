# Взаємодія з сервером (асинхронні екшени)

Після тривалого використання нашого застосунку, ми зрозуміли, що є один серйозний та очевидний мінус — всі дані зберігаються лише на пристрої в локальному сховищі, і немає змоги синхронізуватись із сервером. Тому давайте додамо взаємодію з сервером.

Розглянемо 3 найбільш поширених випадки:

1. Проста взаємодія з сервером — запит на повний список задач.
2. Optimistic update – оновлення даних на фронтенді ще до того, як дані змінились на бекенді.
3. Optimistic create – створення даних (моделі) на фронтенді ще до того, як ми отримали відповідь.

## Проста взаємодія

Отже, розглянемо як реалізовувати асинхронний екшен. В більшій мірі асинхронний екшен — це екшен, який виконується асинхронно (ваш кеп).
Використовуючи файл Арі, який є в папці `imports`, реалізуємо завантаження наших задач з сервера.

Взагалі, в подальшому, старайтесь оголошувати ваші екшени керуючись цими принципом, що екшен повинен бути в тій моделі, до якої він безпосередньо відноситься. Тобто якщо в нас завантаження списку задач, то логічно буде розмістити його в моделі TodoListModel:

```js
import Api from './imports/api';

const TodoListModel = types.model({
  list: types.array(TodoModel),

  // також нам потрібний деякий стейт
  isLoading: false,
})
.views((self) => ({
  /* ...тут наша в'юха get favorites */
}))
.actions((self) => ({
  /* ...тут екшн додавання задачі */

  // асинхронний екшен? логічно, що це має бути екшен, проте асинхронний
  async fetchTodos() {
    try {
      self.isLoading = true;

      const todos = await Api.Todos.fetchList();

      self.list = todos;
      self.isLoading = false;
    } catch (err) {
      self.isLoading = false;
      // якщо буде помилка — просто її виведемо
      console.log(err);
    }
  }
}));

// давайте спробуємо запустити

const rootStore = RootModel.create({});

rootStore.todos.fetchTodos();
```

Проте ми отримуємо отаку помилку:

> Cannot modify 'AnonymousModel@/todos', the object is protected and can only be modified by using an action.

Діло в тому, що коли ми змінюємо `self.isLoading`, ми ще знаходимось в межах екшена, проте як тільки починається асинхронна частина — `await Api.Todos.fetchList()`, MST більше не може слідкувати за змінами які відбувається, відповідно не може їх зібрати в один "batch", тому все що ми виконуємо після — поза екшеном.
Щоб це обійти, нам потрібно створити екшени для всіх змін, в такому випадку все буде відбуватись, використовуючи екшен:

```js
const TodoListModel = types.model({ /* модель */ })
.views((self) => ({ /* ...тут наша в'юха get favorites */ }))
.actions((self) => ({
  /* ...тут екшн додавання задачі */

  async fetchTodos() {
    try {
      // тут ми досі в екшені fetchTodos, тому все працює
      self.isLoading = true;

      const todos = await Api.Todos.fetchList();

      // викликаємо окремий екшен для зміни
      self.fetchTodosSuccess(todos);
    } catch (err) {
      // і тут також
      self.fetchTodosError(err);
    }
  },

  fetchTodosSuccess(todos) {
    self.list = todos;
    self.isLoading = false;
  },

  fetchTodosError(err) {
    self.isLoading = false;
    console.log(err);
  }
}));
```

Проте є кращий (а чи дійсно кращий?) спосіб — використання генераторів та обгортки у вигляді `flow`, яку нам надає MST – така собі корутина:

```js
// імпортуємо flow з mobx-state-tree, не mobx
import { types, flow } from 'mobx-state-tree';

const TodoListModel = types.model({ /* модель */ })
.views((self) => ({ /* ...тут наша в'юха get favorites */ }))
.actions((self) => ({
  /* ...тут екшн додавання задачі */

  // зверніть увагу що це генератор — "function*"
  fetchTodos: flow(function* fetchTodosFlow() {
    try {
      self.isLoading = true;

      // замість `await` використовуємо `yield`
      const todos = yield Api.Todos.fetchList();

      self.list = todos;
      self.isLoading = false;
    } catch (err) {
      self.isLoading = false;
      console.log(err);
    }
  }),
}));

// запускаємо
const rootStore = RootModel.create({});

// flow повертає проміс
rootStore.todos.fetchTodos().then(() => {
  prettyPrint(rootStore);
});
```

Та отримуємо такий результат:

```json
{
  "todos": {
    "list": [
      {
        "id": "4fa51175-9de1-473f-961e-c9f485b0ed4d",
        "title": "Яйця",
        "isCompleted": false,
        "isFavorite": false
      },
      {
        "id": "8c70682f-5aa1-4206-a5dc-6ba25902f916",
        "title": "Помідори",
        "isCompleted": false,
        "isFavorite": false
      }
    ]
  },
  "groups": {
    "list": []
  }
}
```

## Оптимістичні оновлення

Часто UX вимагає того, щоб зміни для користувача відбулись ще до того, як сервер нам щось відповів, тобто ми даємо серверу кредит довіри та оптимістично оновляємо наш фронтенд, і якщо відповідь сервера позитивна — нічого не робимо, якщо ж помилка — відміняємо всі зміни та даємо користувачу зрозуміти, що відбулась помилка і йому треба повторити дію знову.

Давайте реалізуємо такий сценарій для зміни `isCompleted` нашої задачі, додавши асинхронний flow екшен до моделі TodoModel:

```js
const TodoModel = types.model({
 // всі поля моделі

 isTogglingCompleted: false,
 isErrorTogglingCompleted: false,
})
.actions((self) => ({
  // ...решту екшенів

  // а цей екшен ми замінимо із звичайної зміни на асинхронну взаємоді з сервером
  toggleFavorite: flow(function* toggleFavoriteFlow() {
    // зберігаємо попереднє значення
    const oldValue = self.isFavorite;

    // змінюємо значення вже, щоб в інтерфейсі було вже нове
    self.isFavorite = !self.isFavorite;
    // якщо користувач спробував знову, забираємо помилку
    self.isErrorTogglingCompleted = false;
    // показуємо якийсь `loading`
    self.isTogglingCompleted = true;

    try {
      // надсилаємо нове значення на сервер
      yield Api.Todos.setCompleted(self.isFavorite);

      self.isTogglingCompleted = false;
      // і все, ми зробили все що потрібно
    } catch(err) {
      self.isTogglingCompleted = false;
      // показуємо помилку
      self.isErrorTogglingCompleted = true;
      // міняємо значення на попереднє
      self.isFavorite = oldValue;
    }
  }),
}));
```

## Оптимістичне створення моделі

Цей підхід доволі часто можна зустріти в сучасних інтерфейсах, наприклад коли у вас є застосунок для спілкування. Меседж створюється як тільки користувач натиснув кнопку "відправити", проте біля нього показується щось накшталт індикатора про те, що цей меседж ще відправляється на сервер. Ми зробимо щось схоже із нашими туду-елементами. При додаванні ногово, ми відразу ж будемо створювати нову модель, додавати її до списку, і тільки після цього надсилати дані на сервер. 

Найпростіший спосіб реалізувати це — використати [хуки життєвого циклу моделі](https://github.com/mobxjs/mobx-state-tree#lifecycle-hooks-for-typesmodel). Як саме? Алгоритм наших дій буде наступний:

1. Створюємо нову модель, додаємо її в дерево

Для цього нам потрібно не потрібно нічого змінювати, тому що у нас все готово. Проте давайте розглянемо що тут відбувається і для чого ми це робимо ще раз:

```js
const TodoListModel = types.model({
  list: types.array(TodoModel),
})
.actions((self) => ({
  add(title) {
    // створюємо новий об'єкт задачі
    const todo = {
      // і тут поле `id` — тимчасове
      // ми його замінимо пізніше на те, яке прийде з сервера
      id: uuid(),
      title,
    };

    // додаємо в масив
    self.list.unshift(todo);
  }
}));
```

2. В нашу модель Todo додаємо екшен, який буде відправляти її на сервер.

> Зверніть увагу: `identifier` відображає ідентифікатор у вигляді рядку проте в нас може бути випадок, коли ми використовуємо `identifierNumber` який відображає ідентифікатор-число, але значення яке ми генеруємо на клієнті — рядок. Відповідно вам, можливо, краще використати `types.union(types.identifier, types.identifierNumber)`, де `union` означає, що тип може бути або `identifier` або `identifierNumber`, і обидва вони будуть валідними

```js
import { types, flow } from 'mobx-state-tree';

const TodoModel = types.model({
  id: types.identifier,
 // ...решту полів моделі
 // і тут додаємо стан для нового екшену
 isSending: false,
 isErrorSending: false,
})
.actions((self) => ({
  // ...решту екшенів

  // додаємо екшен, який відправить задачу на сервер
  send: flow(function* sendFlow() {
    self.isErrorSending = false;
    // показуємо якийсь `loading`
    self.isSending = true;

    try {
      // надсилаємо нове значення на сервер
      const todo = yield Api.Todos.create(self);
      // оновляєм, мутуючи нашу модель
      // в тому числі заміняємо id на новий
      Object.assign(self, json);
      // забираємо `loading`
      self.isSending = false;
    } catch(err) {
      self.isSending = false;
      // показуємо помилку
      self.isErrorSending = true;
    }
  }),
}));
```

3. На `afterAttach` запускаємо цей екшен:

`afterAttach` – це як `componentDidMount` в реакті — спрацьовує після того, як модель була додана в дерево. Ідеальне місце для того, щоб викликати якусь логіку, яку б ви викликали в `componentDidMount` вашого компонента, чи не так? Це ми і зробимо:

```js
const TodoModel = types.model({
  // всі наші поля моделі
})
.actions((self) => ({
  afterAttach() {
    // просто запускаємо екшен надсилання на сервер
    // який виконає всю необхідну роботу
    self.send();
  },

  // ...решту екшенів

  send: flow(function* sendFlow() {
    // тут надсилання
  }),
}));
```

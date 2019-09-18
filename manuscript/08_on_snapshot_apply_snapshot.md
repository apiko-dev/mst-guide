# onSnapshot та applySnapshot

Особливо уважні могли б зрозуміти, що який би зручний наш застосунок не був (жарт), або ж як добре він свої функції не виконував, при перезавантаженні весь стан втрачається, так як ми його нікуди не зберігаємо. Для того щоб мати завжди актуальні дані, нам потрібно оновляти кеш при кожній зміні моделі.

mobx-state-tree надає нам змогу підписатись на зміни моделі, використовуючи `onSnapshot`, а також застосовувати до неї будь який снепшот, який пройде валідацію за допомогою `applySnapshot`.

Для того щоб вирішити проблему втрати стану, ми напишемо невеличкий аналог `redux-persist`:

```js
import { applySnapshot, onSnapshot } from 'mobx-state-tree';

const PERSIST_KEY = 'persist';
// приймаємо екземпляр нашого стору (моделі), який ми хочемо зберігати
// а також якийсь storage, який реалізовує інтерфейс local-forage
function createPersist(store, storage) {
  // дозволить нам відновлювати наш кеш, застосовувати снепшот
  // до стану нашого стору
  function rehydrate() {
    const snapshot = storage.getItem(PERSIST_KEY);

    if (snapshot) {
      applySnapshot(store, JSON.parse(snapshot));
    }
  }

  // дозволить нам очищати наш кеш
  function purge() {
    storage.removeItem(PERSIST_KEY);
  }

  // так як колбек onSnapshot викликається кожного разу коли змінюється модель
  // це ідеально підходить для того щоб завжди зберігати найактуальніші дані
  onSnapshot(store, (snapshot) => {
    storage.setItem(PERSIST_KEY, JSON.stringify(snapshot));
  });

  return {
    purge,
    rehydrate,
  };
}

// створюємо наш персіст
// він після цього ж починає писати всі снепшоти в сторедж
const persist = createPersist(rootStore, localForage);

// викликаємо регідрейт для того, щоб відновити стан нашого стору до того стану,
// який ми востаннє записали в сторедж
persist.rehydrate();
```

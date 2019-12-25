---
title: “Bug-O” Нотація
date: '2019-01-25'
spoiler: Яка 🐞(<i>n</i>) вашого API?
---

Пишучи чутливий до продуктивності код варто пам'ятати про його алгоритмічну складність. Вона часто виражається за допомогою [Big-O нотації](https://rob-bell.net/2009/06/a-beginners-guide-to-big-o-notation/).

Big-O - це міра того, **як швидко буде сповільнюватись робота вашого коду з ростом кількості вхідних даних**. Наприклад, якщо алгоритм сортування має складність O(<i>n<sup>2</sup></i>), сортування в 50 разів більше елементів приблизно буде в 50<sup>2</sup> = 2,500 разів повільніше. Big O не дає вам точного числа, але вона допомагає вам зрозуміти як алгоритм *масштабується*.

Декілька прикладів: O(<i>n</i>), O(<i>n</i> log <i>n</i>), O(<i>n<sup>2</sup></i>), O(<i>n!</i>).


В будь якому випадку, **цей пост не про алгоритми чи ефективність**. Він про API та дебаг. Виявляється, дезайн API включає схожі міркування.

---

Значна частина часу марнується на пошук і фікс помилок у коді. Більшість девів хотіли б знаходити баги швидше. І як би приємно не було знаходити важливий баг, це відстій тратити цілий день на пошук єдиного багу замість виконання наступного етапу вашого роадмапа.

Досвід у відлагодженні впливає на вибір абстракцій, бібліотек і інструментів. Деякі API і мовні конструкції роблять цілий клас помилок неможливими. А деякі - генерують безкінечну їх кількість. **Але як можна сказати які з них які?**

Багато онлайн-обговорень API в першу чергу стосуються естетики. Але естетика [не дає відповіді](/optimized-for-change/) на питання як приємно користуватись API на практиці.

**І я маю метрику, яка допомагає думати про це. Я називаю її *Bug-O* нотація:**

<font size="40">🐞(<i>n</i>)</font>

Big-O описує як алгоритм сповільнюється з ростом кількості вхідних даних. Тоді як *Bug-O* показує як API сповільнює *вашу роботу* з ростом кодової бази.

---

Наприклад, розглянемо наступний код, який вручну оновлює DOM імперативними операторами `node.appendChild()`, `node.removeChild()` і не має чіткої структури:

```jsx
function trySubmit() {
  // Сегмент 1
  let spinner = createSpinner();
  formStatus.appendChild(spinner);
  submitForm().then(() => {
  	// Сегмент 2
    formStatus.removeChild(spinner);
    let successMessage = createSuccessMessage();
    formStatus.appendChild(successMessage);
  }).catch(error => {
  	// Сегмент 3
    formStatus.removeChild(spinner);
    let errorMessage = createErrorMessage(error);
    let retryButton = createRetryButton();
    formStatus.appendChild(errorMessage);
    formStatus.appendChild(retryButton)
    retryButton.addEventListener('click', function() {
      // Сегмент 4
      formStatus.removeChild(errorMessage);
      formStatus.removeChild(retryButton);
      trySubmit();
    });
  })
}
```

Проблема цього коду не в тому, що він “потворний”, тому що ми все ще не розмовляємо про естетику. **Якщо тут є баг, то я не знаю звідки починати пошук.**

**Взалежності від того в якому порядку викликаються колбеки та відбуваються івенти, кількість можливих напрямків виконання коду породжує комбінаторний вибух.** В деяких з них, я побачу вірні повідомлення. В інших же, я побачу повторюємі спінери, повідомлення про помилки і, можливо, збої.

Дана функція має 4 різні секції і ніяких гарантій щодо їх послідовності. Мої дуже ненаучні підрахунки показують, що тут 4×3×2×1 = 24 різних напрямків виконання кода. І якщо додати ще 4 сегменти, матимо 8×7×6×5×4×3×2×1 — *сорок тисяч* комбінацій. Що ж, бажаю успіхів.

**Іншими словами, Bug-O цього коду - 🐞(<i>n!</i>)** , де *n* - це кількість сегментів кода, що стосуються DOM. Так, це *факторіал*. Звичайно, це не дуже научний підхід. Не всі переходи між сегментами можливі на практиці. Але з іншого боку, кожен сегмент може відпрацювати більше одного разу. <span style="word-break: keep-all">🐞(*¯\\\_(ツ)\_/¯*)</span> все ще дуже погано. Можна краще.

---

Для покращення Bug-O цього коду, ми можемо обмежити кількість можливих станів і результатів. Для цього не потрібна ніяка бібліотека. Ми просто жорстко задамо структуру кода. Як це може виглядати наприклад:

```jsx
let currentState = {
  step: 'initial', // 'initial' | 'pending' | 'success' | 'error'
};

function trySubmit() {
  if (currentState.step === 'pending') {
    // Не дозволяємо повторних submit-ів
    return;
  }
  setState({ step: 'pending' });
  submitForm().then(() => {
    setState({ step: 'success' });
  }).catch(error => {
    setState({ step: 'error', error });
  });
}

function setState(nextState) {
  // Видаляємо всіх потомків
  formStatus.innerHTML = '';

  currentState = nextState;
  switch (nextState.step) {
    case 'initial':
      break;
    case 'pending':
      formStatus.appendChild(spinner);
      break;
    case 'success':
      let successMessage = createSuccessMessage();
      formStatus.appendChild(successMessage);
      break;
    case 'error':
      let errorMessage = createErrorMessage(nextState.error);
      let retryButton = createRetryButton();
      formStatus.appendChild(errorMessage);
      formStatus.appendChild(retryButton);
      retryButton.addEventListener('click', trySubmit);
      break;
  }
}
```

Цей код не дуже відрізняється. Крім того, він більш багатослівний. Але його *набагато* простіше дебажити в тому числі через наступний участок коду:

```jsx{3}
function setState(nextState) {
  // Видаляємо всіх потомків
  formStatus.innerHTML = '';

  // ... код додавання елементів до formStatus...
```

Коли ми очищуємо статус форми перед проведенням будь яких маніпуляцій, ми забезпечуемо те, що робота з DOM завжди починається з чистого листа. Це спосіб боротьби з [єнтропією](/the-elements-of-ui-engineering/) — ми не дозволяємо помилкам накопичуватись. Це еквівалент “вимкнути та увімкнути”, і такий підхід працює прекрасно.

**Якщо ми зустрічаємо помилку, нам потрібно повернутись на *крок* назад — до попереднього виклику `setState`.** Bug-O в такому випадку складає 🐞(*n*) де *n* - це кількість зарендерених сегментів кода. Тепер складність дебага дорівнює чотирьом (тому що ми маємо чотири варіанти в `switch`).

Ми все ще маємо [стан гонитви](https://uk.wikipedia.org/wiki/%D0%A1%D1%82%D0%B0%D0%BD_%D0%B3%D0%BE%D0%BD%D0%B8%D1%82%D0%B2%D0%B8) в управлінні станом. Але дебажити тепер простіше тому, що кожен проміжний стан може бути залогований та проінспектований. Крім того, ми явно можемо заборонити деякі переходи:

```jsx
function trySubmit() {
  if (currentState.step === 'pending') {
    // Не дозволяємо повторних submit-ів
    return;
  }
```

Звичайно, кожного разу обнуляти DOM - це компроміс. Наївно видаляючи і додаючи елементи DOM кожного разу руйнує їх внутрінній стан, заставляє втратити фокус і спричиняє жахливі наслідки у великих программах.

Саме тому такі бібліотеки як React можуть бути корисні. Вони дозволяють вам *думати* в ключі кожного разу створення UI з нуля без реалізації цього функціоналу:

```jsx
function FormStatus() {
  let [state, setState] = useState({
    step: 'initial'
  });

  function handleSubmit(e) {
    e.preventDefault();
    if (state.step === 'pending') {
      // Не дозволяємо повторних submit-ів
      return;
    }
    setState({ step: 'pending' });
    submitForm().then(() => {
      setState({ step: 'success' });
    }).catch(error => {
      setState({ step: 'error', error });
    });
  }

  let content;
  switch (state.step) {
    case 'pending':
      content = <Spinner />;
      break;
    case 'success':
      content = <SuccessMessage />;
      break;
    case 'error':
      content = (
        <>
          <ErrorMessage error={state.error} />
          <RetryButton onClick={handleSubmit} />
        </>
      );
      break;
  }

  return (
    <form onSubmit={handleSubmit}>
      {content}
    </form>
  );
}
```

Код виглядає трішки по іншому, проте принцип залишився тим самим. Абстракція компонента примушує додержуватись границь, так ви знаєте, що *інший* код сторінки не *зламає* щось всередині компонента(його DOM або state). Підхід компонентів дозволяє зменшити Bug-O.

Фактично, якщо *будь яке* значення в DOM React-додатка виглядає помилковим ви завжди можете відслідити звідки воно прийшло заглянувши в код батьківського компонента в дереві React один за одним. Незалежно від розміру додатку складність трасування помилкового значення складає 🐞(*висота React дерева*).

**В наступний раз коли ви побачите дискусію щодо API, подумайте: а яка 🐞(*n*) дебагу середньої помилки цього API?** Щодо вже існуючих API і принципів, з якими ви добре знайомі? Redux, CSS, наслідування — всі вони мають власний Bug-O.

---
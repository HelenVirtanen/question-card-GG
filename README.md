# QuestionCard

## Контекст
Контент вопроса приходит как JSON (TipTap), формулы через KaTeX.  
Пользователь кликает быстро, API иногда косячит.

## Часть 1. Архитектура

### Структура QuestionCard

QuestionCard  
├── QuestionStem (TipTapRenderer + KaTeX)  
│     └─ отображает текст вопроса с формулами  
├── AnswerOptions  
│     └─ AnswerOption (radio/button) отображает ответы и выбор ответа  
├── ActionBar  
│     ├─ CheckAnswerButton  
│     └─ Disabled если !selectedAnswer или уже checked  
├── Explanation (conditional)  
│     └─ показывается только после check  
└── DemoOverlay (conditional)  
      └─ скрывает explanation и показывает CTA на оплату  

## State management:
1) **selectedAnswer** - хранится локально в QuestionCard, передаётся в AnswerOptions как controlled state   
- проще сбрасывать state при смене questionId
- вся логика вопроса в одном месте

2) **isChecked** - в QuestionCard, влияет на отображение Explanation и блокировку кнопки Check
3) **questionId** - передается через props;  
При смене вопроса сбрасывает **selectedAnswer** и **isChecked**  

4) **isChecking** — защищает от быстрых кликов, используется для loading-состояния кнопки
**Быстрые клики** - нужно блокировать CheckAnswerButton **до** завершения API call (если есть), иначе состояние не ломается.
5) **isCorrect** — результат проверки, нужен для визуальной обратной связи
6) **showExplanation** — управляет показом explanation после check


## Часть 2. Псевдокод логики: 
**Состояние в QuestionCard**
```
// questionId приходит через props и не хранится в локальном state

state:
  selectedAnswer = null   // id выбранного ответа
  isChecked = false
  isChecking = false  // защита от быстрых кликов
  isCorrect = null   // true/false после проверки
  showExplanation = false

// derived
isCheckDisabled = !selectedAnswer || isChecked || isChecking
```

**Выбор ответа:**
```
onSelectAnswer(answerId):
  if (isChecked) return   // после check нельзя менять ответ
  selectedAnswer = answerId

```

**Проверка ответа:**
```
onCheckAnswer():
  if (!selectedAnswer || isChecked) return

  isChecking = true

  call checkAnswerAPI(selectedAnswer)
    .then(result):
      isCorrect = result.isCorrect // для визуальной подсветки
      isChecked = true
      showExplanation = true
    .catch():
      showError("Не удалось проверить ответ, попробуйте ещё раз")
      isChecked = false
    .finally():
      isChecking = false
```

**Смена вопроса**
```
onQuestionChange(questionId):
  resetState()

resetState():
  selectedAnswer = null
  isChecked = false
  isChecking = false
  showExplanation = false
```

**Рендеринг**
```
render():
  render QuestionStem

  render AnswerOptions
    disabled = isChecked

  render CheckAnswerButton
    disabled = isCheckDisabled
    loading = isChecking

  if (isChecked):
    render Explanation
```

### Особенности: 
- explanation никогда не показывается до check;
- состояние обязательно сбрасывается при смене questionId;
- быстрые клики не ломают UI из-за isChecking;
- после check пользователь не может менять ответ;
- disabled состояния вычисляются явно, а не «магией».


## Часть 3. Edge cases и UX
1. **explanation отсутствует**
- после check показывается *fallback*: «Объяснение к этому вопросу пока недоступно»
- UI не ломается, пустых блоков нет
- кнопка Check работает *как обычно*

2. **В stem только формулы (KaTeX)**
- контейнер QuestionStem имеет **минимальную высоту**
- формулы центрируются **по вертикали**
- **Нет лишних отступов**, чтобы карточка не выглядела «пустой»

3. *В stem очень длинный текст*
- контент скроллится внутри QuestionStem
- максимальная высота ограничена (чтобы кнопки всегда были видны)
- текст не выносит ActionBar за пределы экрана (position: sticky/fixed)

4. **KaTeX упал с ошибкой**
- ошибка *ловится внутри* QuestionStem
- показывается *fallback*: «Не удалось отобразить формулу»
- остальной UI (ответы, check) *остаётся рабочим*, пользователь может повторно нажать check

5. **Пользователь меняет ответ после check**
- AnswerOptions становятся *disabled*
- визуально показывается *выбранный* ответ
- *нет пересчёта* результата, состояние остаётся стабильным

6. **Пользователь в demo-режиме**
Explanation скрыт/заблюрен, тогда: 
- показывается текст: «Объяснение доступно в полной версии»
- явный CTA: кнопка «Открыть полный доступ»
- при клике на СТА, после оплаты пользователь возвращается к Explanation
Результат: пользователь понимает, почему контент недоступен и что нужно сделать, чтобы получить доступ, сразу видит результат выполненных условий  
* В demo-режиме можно показывать правильный ответ без explanation для мотивации.

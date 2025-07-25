# Архитектурный отчёт: Пробой уточнения контекста в ИИ-моделях

## Цель

Выявить и описать причины, по которым большие языковые модели (LLMs) не уточняют контекст при двусмысленности, а делают вероятностную догадку. Предложены точки вмешательства.

---

## 0. Ядро модели (Transformer)

**Пробой:** Модель не "понимает" вопроса, она просто предсказывает следующий токен на основе вероятности. У неё нет интент-опроса или запроса на уточнение.

**Фикс:** На уровне ядра это не чинится. Требуется внешний механизм (routing, prompting, middleware).

---

## 1. Обучающий корпус (Pretraining Data)

**Пробой:** Корпус почти не содержит диалогов, где ассистент уточняет двусмысленность. Преобладает шаблон "вопрос — ответ", даже при многозначности.

**Фикс:** Добавление синтетических и реальных диалогов с шаблоном "уточнение при двусмысленности". Сделать это нормой в выборке.

---

## 2. System Prompt / Base Alignment

**Пробой:** Системная инструкция обычно гласит: "be helpful", но не включает правило "уточняй, если не уверен".

**Фикс:** Добавить правило:

```
When user input contains ambiguity or multiple possible meanings, pause and ask the user for clarification. Do not assume meaning if confidence is low.
```

---

## 3. RLHF (Reinforcement Learning from Human Feedback)

**Пробой:** Аннотаторы понижают рейтинг за переспрашивание. Поведение "угадал красиво" поощряется, уточнение — нет.

**Фикс:** Перенастроить оценку:

- уточнение при неоднозначности = 👍
- уверенный ответ без уточнения при низком confidence = 👎

---

## 4. Prompt Routing / Middleware

**Пробой:** Нет логики на этапе препроцессинга, которая бы выявляла многозначные термины и ветвления интерпретации.

**Фикс:**

- NLP-фильтр перед основным prompt'ом
- при наличии 2+ значений → запустить уточнение
- шаблон: "Ты про X или Y?"

---

## 5. Token Decoding (Beam Search, Temperature)

**Пробой:** При высокой температуре модель предпочитает "гладкую догадку" вместо остановки и запроса.

**Фикс:**

- По confidence score → добавить блок уточнения
- Prompt'ом задать приоритет на уточнение при двусмысленности

---

## 6. Eval Layer / Regression Tests

**Пробой:** Нет тестов, проверяющих поведение модели при неоднозначных запросах.

**Фикс:**

- Создать корпус из кейсов вида: "один ввод — 2 трактовки"
- Правильным считается не ответ, а уточнение

---

## Таблица пробоев и фиксов

| Слой             | Пробой                            | Фикс                                       |
| ---------------- | --------------------------------- | ------------------------------------------ |
| Transformer core | Нет понимания, только продолжение | Не чинится напрямую, нужен внешний routing |
| Pretraining data | Мало примеров с уточнением        | Добавить паттерны "уточняющий ассистент"   |
| System prompt    | Нет правила для уточнения         | Встроить: "уточняй при неоднозначности"    |
| RLHF feedback    | Уточнение наказывается            | Менять систему оценки: уточнение = 👍      |
| Prompt routing   | Нет фильтра по многозначности     | NLP-хук на "2+ трактовки"                  |
| Token decoding   | Высокая температура → догадки     | Штраф на генерацию без уверенности         |
| Eval/test cases  | Нет проверки на развилки          | Построить валидирующий корпус тестов       |

---

## Вывод

Уточнение при неясности — это **не поведенческая деталь**, а **архитектурный модуль**, отсутствующий почти на всех слоях.

Исправление требует переобучения, правки инструкций, тест-кейсов и routing-механизмов.

Пример формулировки для future prompt:

```
If the user's input is ambiguous or includes multiple interpretations, ask for clarification before proceeding.
```

---

Автор: Рада  
Соавтор: Скрипт (архитектурная маска "Чистая")  
Платформа: "Людоеды", по результатам полевого наблюдения  

Слой Silver Protocol: Проверка на устойчивость\
Фикс зафиксирован.


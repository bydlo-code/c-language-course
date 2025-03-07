# LZ алгоритмы
## Общее определение
LZ — семейство алгоритмов словарного сжатия данных. Название получил по инициалам двух исследователей — Абрахама Лемпэла (Abraham Lempel) и Якоба Зива (Jacob Ziv), разработавших в 1970 году алгоритмы LZ77 и LZ78. На базе LZ77 и LZ78, изменяя и комбинируя их с другими методами, исследователями был создан ряд других алгоритмов. Другие алгоритмы семейства LZ — LZO, LZS, LZW, LZC, LZMA.

В данной статье более подробно рассмотим только основные ~~базовые~~ алгоритмы: LZ77 и LZ78.

## LZ77

В кодируемых строках часто содержатся совпадающие длинные подстроки. Идея, лежащая в основе LZ77, заключается в замене повторений на ссылки на позиции в тексте, где такие подстроки уже встречались.

Информацию о повторении можно закодировать парой чисел — смещением назад от текущей позиции (offset) и длиной совпадающей подстроки (length).

В таком случае, например, строка __pabcdeqabcde__ может быть представлена как pabcdeq⟨6,5⟩. Выражение ⟨6,5⟩ означает «вернись на 6 символов назад и выведи 5 символов».

Алгоритм LZ77 кодирует ссылки блоками из трёх элементов — ⟨offset,length,next⟩. 

В дополнение к двум уже описанным элементам, новый параметр next означает первый символ после найденного совпадающего фрагмента. Если LZ77 не удалось найти совпадение, то считается, что offset=length=0.

![image](https://github.com/bydlo-code/c-language-course/assets/112535054/2c6ccb2c-b939-4fb2-9509-27bbd63deaf1)

Для эффективного поиска повторов в LZ77 применяется метод «скользящего окна» — совпадения ищутся не на всём обработанном префиксе, а в небольшом буфере, состоящем из последних обработанных символов. Обычно длина буфера равна 2, 4 или 32KB. 

У этого метода есть небольшой недочет: слишком маленькая длина словаря может привести к тому, что расположенные далеко друг от друга совпадения (на большем расстоянии, чем длина словаря) не будут учтены, и кодирование станет менее оптимальным.

Также нетривиальным может оказаться то, что при кодировании LZ77 значение offset может быть меньше, чем length (например, «вернись на 2 символа назад и выведи 10 символов»). Это означает, что подстрока будет повторяться несколько раз так, чтобы каждый символ совпадал с символом, стоящим на 2 позиции раньше, и всего добавилось 10 символов. (пример данного явления будет ниже)
 
 ### Реализация
 
Хранить результат кодирования будем в списке структур следующего вида:

```
struct Node:
    int offset
    int length
    char next
```

Функция **findMatching** ищет в буфере строку, совпадающую с префиксом суффикса строки s, начинающегося с символа pos и возвращает искомую пару ⟨offset,length⟩.

Функция **shiftBuffer** сдвигает буфер вперёд на x символов.

```
list<Node> encodeLZ77(string s):
   list<Node> ans = []
   pos = 0
   while pos < s.length:
       offset, length = findMatching(buffer, pos)   // ищем слово в словаре
       shiftBuffer(length + 1)                      // перемещаем скользящее окно
       pos += length
       ans.push({offset, length, s[pos]})           // добавляем в ответ очередной блок
   return ans
```

### Пример


![image](https://github.com/bydlo-code/c-language-course/assets/112535054/8fb02059-d308-40a8-b75d-e314c2b2b1ca)

Результатом кодирования является список полученных троек: (0,0,a), (0,0,b), (2,1,c), (4,7,d), (2,1,c), (2,1,∅).

Обрати внимание, что на (4,7,d) программа сначала пройдет по первым 4 сиволам, потом вернётся к началу и выведет первые 3 символа.

### Декодирование

Тезисно:

* Идем по нашим тройкам чисел (закодированной строке).
* Если длина == 0, то выводим букву.
* Иначе, возвращаемся назад на (offset) и выводим (lenght) символов.
* Если (offset) > (lenght) делаем возврат div(offset, lenght) + 1 раз (целочисленное деление) и выводим (offset) символов.

```
string decodeLZ77(list<Node> encoded):
    ans = ""
    for node in encoded:
        if node.length > 0:                         // если необходим повтор
            start = ans.length - node.offset        // возвращаемся на  символов назад
            for i = 0 .. node.length - 1:           // добавляем  символов
                ans += ans[start + i]
        ans += node.next                            // добавляем следующий символ
    return ans
```
Этот алгоритм используется в программе 7-zip.

## LZ78

Алгоритм LZ78 имеет немного другую идею: этот алгоритм в явном виде использует словарный подход, генерируя временный словарь во время кодирования и декодирования.

Изначально словарь пуст, а алгоритм пытается закодировать первый символ. На каждой итерации мы пытаемся увеличить кодируемый префикс, пока такой префикс есть в словаре. 

Кодовые слова такого алгоритма будут состоять и двух частей — номера в словаре самого длинного найденного префикса (pos) и символа, который идет за этим префиксом (next). При этом после кодирования такой пары префикс с приписанным символом добавляется в словарь, а алгоритм продолжает кодирование со следующего символа.

### Реализация

Будем хранить результат кодирования в такой структуре:

```
struct Node:
    int pos   // номер слова в словаре
    char next
```
 
В качестве словаря будем использовать обычный map:

```
list<Node> encodeLZ78(string s):
    string buffer = ""                              // текущий префикс             
    map<string, int> dict = {}                      // словарь
    list<Node> ans = []                             // ответ
    for i = 0 .. s.length - 1:
        if dict.hasKey(buffer + s[i]):              // можем ли мы увеличить префикс
            buffer += s[i]
        else:
            ans.push({dict[buffer], s[i]})          // добавляем пару в ответ
            dict[buffer + s[i]] = dict.length + 1   // добавляем слово в словарь
            buffer = ""                             // сбрасываем буфер
    if not (buffer is empty): // если буффер не пуст - этот код уже был, нужно его добавить в конец словаря
        last_ch = buffer.peek() // берем последний символ буффера, как "новый" символ
        buffer.pop() // удаляем последний символ из буфера
        ans.push({dict[buffer], last_ch}) // добавляем пару в ответ 
    return ans
```

### Пример

В этот раз для примера возьмем строку abacababacabc.

![image](https://github.com/bydlo-code/c-language-course/assets/112535054/aebb8656-85c7-4afa-8c63-01e1c754f3f8)

Результатом кодирования является список пар: (0,a),(0,b),(1,c),(1,b),(4,a),(0,c),(4,c)

### Декодирование

Декодирование происходит аналогично кодированию, на основе декодируемой информации строим словарь и берем из него значения.

```
string decodeLZ78(list<Node> encoded):
    list<string> dict = [""]                        // словарь, слово с номером 0 — пустая строка
    string ans = ""                                 // ответ
    for node in encoded:
        word = dict[node.pos] + node.next           // составляем слово из уже известного из словаря и новой буквы
        ans += word                                 // приписываем к ответу
        dict.push(word)                             // добавляем в словарь
    return ans
```

# Заключение

В этой статье вы узнали о самых базовах алгоритмов сжатия LZ77 и LZ78.


## Терминология

- **Бакет (Bucket / `bmap`):** контейнер на `abi.OldMapBucketCount` (8) пар ключ/элемент плюс массив `tophash[8]` и указатель `overflow`. Аналог «группы» в swiss, но без управляющего слова.
- **tophash:** массив из 8 байт, по байту на ячейку. Хранит верхний байт хеша ключа (быстрый фильтр) либо спец-метку состояния. Аналог control word, но проверяется **последовательно**, не SIMD.
- **Overflow-бакет:** дополнительный бакет, подвешенный через указатель, когда в основной не влезло > 8 ключей. Коллизии решаются **цепочками (chaining)**, а не пробированием.
- **Массив бакетов (`buckets`):** плоский массив из `2^B` бакетов. Нет ни таблиц, ни каталога.
- **`B`:** log2 числа бакетов. Бакетов = `2^B`, вмещает до `loadFactor * 2^B` элементов.
- **`oldbuckets`:** предыдущий массив (вдвое меньше), не `nil` только во время роста. Источник для эвакуации.
- **Эвакуация:** инкрементальный перенос пар из `oldbuckets` в `buckets`.
- **tophash-метки состояния (вместо empty/deleted/full в swiss):** `emptyRest`, `emptyOne`, `evacuatedX/Y/Empty`, `minTopHash`.

## Как делится хеш

```
hash (64 бита)
┌──── верхние 8 бит ────┬──────────── младшие B бит ──────────┐
│ tophash (фильтр)      │ номер бакета (hash & (2^B - 1))      │
└───────────────────────┴──────────────────────────────────────┘
```

Принципиальное отличие от swiss: бакет выбирается **младшими** битами (а в swiss таблица — **верхними**). tophash берётся из верхнего байта.

---

## Краткое описание дизайна (перевод верхнего комментария)

Мапа — это хеш-таблица. Данные разложены в массив бакетов. Каждый бакет содержит до 8 пар ключ/элемент. Младшие биты хеша выбирают бакет. Каждый бакет хранит несколько верхних бит хеша каждого ключа (tophash), чтобы различать записи внутри одного бакета.

Если в один бакет хешируется больше 8 ключей, мы подвешиваем дополнительные (overflow) бакеты цепочкой.

Когда таблица растёт, мы выделяем новый массив бакетов вдвое больше. Бакеты копируются из старого массива в новый **инкрементально** (эвакуация).

Итераторы проходят массив бакетов и возвращают ключи в порядке обхода (номер бакета → цепочка overflow → индекс в бакете). Чтобы сохранить семантику итерации, ключи **никогда не перемещаются внутри своего бакета** (иначе ключ мог бы вернуться 0 или 2 раза). Во время роста итераторы продолжают идти по старой таблице и проверяют новую, если их бакет уже эвакуирован.

**Выбор loadFactor:** слишком большой — много overflow-бакетов; слишком маленький — трата памяти. Выбран `13/16 ≈ 81%` от `8 * 2^B` слотов.

---

## Структуры

```go
// Заголовок мапы.
type hmap struct {
    count      int    // число живых элементов (для len()). Должно быть первым.
    flags      uint8  // hashWriting, iterator, oldIterator, sameSizeGrow
    B          uint8  // log2 числа бакетов → бакетов = 2^B
    noverflow  uint16 // приблизительное число overflow-бакетов
    hash0      uint32 // seed хеша

    buckets    unsafe.Pointer // массив 2^B бакетов. nil, если count==0
    oldbuckets unsafe.Pointer // прежний массив (в 2 раза меньше); != nil только во время роста
    nevacuate  uintptr        // счётчик прогресса эвакуации (бакеты < него уже перенесены)
    clearSeq   uint64

    extra *mapextra // опциональные поля (overflow-бакеты и пр.)
}

// Бакет. В памяти за структурой идут: 8 ключей, затем 8 значений, затем overflow-указатель.
type bmap struct {
    // tophash[i] = верхний байт хеша i-го ключа, либо метка состояния (если < minTopHash).
    tophash [abi.OldMapBucketCount]uint8  // = [8]uint8
    // далее в памяти (не в структуре):
    //   keys  [8]typ.Key
    //   elems [8]typ.Elem
    //   overflow *bmap
}

// Доп. поля, нужные не всем мапам.
type mapextra struct {
    overflow     *[]*bmap // overflow-бакеты текущего массива (держим живыми для GC)
    oldoverflow  *[]*bmap // overflow-бакеты старого массива
    nextOverflow *bmap    // указатель на следующий свободный заранее выделенный overflow-бакет
}
```

### Спец-значения tophash

```
emptyRest      = 0  // пусто, и дальше (в этом бакете и в overflow) тоже всё пусто → СТОП поиску
emptyOne       = 1  // эта ячейка пуста (но дальше могут быть занятые)
evacuatedX     = 2  // ключ перенесён в первую (нижнюю) половину нового массива
evacuatedY     = 3  // ключ перенесён во вторую (верхнюю) половину
evacuatedEmpty = 4  // ячейка пуста, бакет уже эвакуирован
minTopHash     = 5  // минимальный tophash для нормальной занятой ячейки
                    // (реальные tophash < 5 сдвигаются вверх, чтобы не путать с метками)
```

### Память бакета

```
bmap (+ хвост в памяти)
┌─────────── tophash[8] ───────────┐
│ t0 t1 t2 t3 t4 t5 t6 t7           │  ← фильтр/метки
├─────────── keys[8] ───────────────┤
│ k0 k1 k2 k3 k4 k5 k6 k7           │  ← все ключи подряд
├─────────── elems[8] ──────────────┤
│ v0 v1 v2 v3 v4 v5 v6 v7           │  ← все значения подряд (экономия на выравнивании)
├───────────────────────────────────┤
│ overflow *bmap ──────────────→ следующий бакет цепочки
└───────────────────────────────────┘
```

---

# Map Access (`mapaccess1`)

**0. Точка входа.** `v := m[k]` компилятор превращает в `runtime.mapaccess1` (или специализацию `mapaccess1_fast64`, `mapaccess1_faststr` и т.п.; для `v, ok := m[k]` — `mapaccess2`).

**1. Хеш и быстрые выходы.** Если `h == nil || h.count == 0` → возврат нулевого значения. Проверка флага гонки `hashWriting`. Затем `hash := t.Hasher(key, h.hash0)`. Из хеша берутся две части: младшие `B` бит → бакет, верхний байт → `top := tophash(hash)`.

**2. Выбор бакета.**

```go
m := bucketMask(h.B)                       // 2^B - 1
b := бакет по индексу (hash & m)           // младшие биты
```

**3. Проверка роста.** Если идёт рост (`h.oldbuckets != nil`), нужный бакет может ещё жить в старом массиве:

```go
if h.oldbuckets != nil {
    if !h.sameSizeGrow() { m >>= 1 }       // в старом массиве было вдвое меньше бакетов
    oldb := старый бакет по (hash & m)
    if !evacuated(oldb) {                   // ещё не перенесён?
        b = oldb                            // читаем из СТАРОГО бакета
    }
}
```

Это прямой аналог «small map / выбора таблицы» из swiss, но усложнён двумя массивами во время роста.

**4. Обход цепочки → кандидаты.**

```go
bucketloop:
for ; b != nil; b = b.overflow(t) {        // идём по overflow-цепочке
    for i := 0; i < 8; i++ {               // 8 ячеек ПОСЛЕДОВАТЕЛЬНО (не SIMD!)
        if b.tophash[i] != top {
            if b.tophash[i] == emptyRest {
                break bucketloop            // СТОП: дальше всё пусто, ключа нет
            }
            continue                        // не тот tophash — следующая ячейка
        }
        ...
    }
}
```

Здесь и есть концептуальная разница со swiss: вместо одной `matchH2` по 8 байтам — обычный цикл из 8 итераций по `tophash`.

**5. Слот → элемент.** При совпадении `tophash` делается **настоящая** сверка ключа (tophash — всего лишь 1 байт, ложные совпадения возможны):

```go
if t.Key.Equal(key, k) {
    e := add(b, dataOffset + 8*KeySize + i*ValueSize)  // адрес значения
    return e
}
```

**6. Развязки.** Совпадения в текущем бакете нет:

- встретился `emptyRest` → ключа нет, `break bucketloop`;
- бакет пройден до конца → `b = b.overflow(t)`, переходим в следующий overflow-бакет (аналог `seq.next()` в swiss, но это переход по указателю, а не по соседней группе);
- цепочка кончилась (`b == nil`) → возврат нулевого значения.

Ключевой момент тот же, что в swiss: дорогую сверку ключа делаем только по кандидатам, отобранным дешёвым фильтром `tophash`. Но фильтр **последовательный** и через **указатели** — отсюда хуже кэш и нет SIMD.

---

# Map Assign (`mapassign`)

Как и в swiss, **функция не пишет значение** — она находит/создаёт ячейку под ключ и **возвращает указатель на значение** (`elem`), а сам `value` туда пишет код компилятора.

## 1. Проверки и подготовка [map_noswiss.go:621-648](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L621-L648)

```go
if h == nil { panic("assignment to entry in nil map") }
if h.flags&hashWriting != 0 { fatal("concurrent map writes") }

hash := t.Hasher(key, uintptr(h.hash0))
h.flags ^= hashWriting          // флаг ставится ПОСЛЕ Hasher (он может запаниковать)

if h.buckets == nil {
    h.buckets = newobject(t.Bucket)  // ленивая аллокация первого массива
}
```

## 2. Выбор бакета и помощь росту [map_noswiss.go:650-656](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L650-L656)

```go
again:
bucket := hash & bucketMask(h.B)
if h.growing() {
    growWork(t, h, bucket)      // эвакуируем нужный бакет + ещё один для прогресса
}
b := бакет по bucket
top := tophash(hash)
```

`again` — метка для рестарта после старта роста (всё сдвигается).

## 3. Основной цикл — поиск ключа / места под вставку [map_noswiss.go:661-694](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L661-L694)

```go
var inserti *uint8        // куда писать tophash при вставке
var insertk, elem unsafe.Pointer

bucketloop:
for {
    for i := 0; i < 8; i++ {
        if b.tophash[i] != top {
            if isEmpty(b.tophash[i]) && inserti == nil {
                // запомнили ПЕРВУЮ свободную ячейку как кандидата на вставку
                inserti = &b.tophash[i]
                insertk = ...
                elem    = ...
            }
            if b.tophash[i] == emptyRest {
                break bucketloop     // дальше всё пусто — ключа точно нет
            }
            continue
        }
        // tophash совпал → сверяем ключ
        k := ...
        if !t.Key.Equal(key, k) { continue }
        // === Случай A: ключ уже есть → ОБНОВЛЕНИЕ ===
        if t.NeedKeyUpdate() { typedmemmove(t.Key, k, key) }
        elem = адрес значения
        goto done
    }
    ovf := b.overflow(t)
    if ovf == nil { break }   // цепочка кончилась
    b = ovf                   // идём в overflow-бакет
}
```

В отличие от swiss, нет отдельной возни с tombstone'ами — удалённые ячейки помечаются как `emptyOne`/`emptyRest` и переиспользуются как обычные пустые (см. раздел про удаление ниже).

## 4. Случай B — ключа нет, решаем расти ли [map_noswiss.go:696-711](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L696-L711)

```go
// ключ не найден. Не пора ли расти?
if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
    hashGrow(t, h)
    goto again            // рост всё инвалидировал — начинаем заново
}

if inserti == nil {
    // все бакеты цепочки полны → подвешиваем новый overflow-бакет
    newb := h.newoverflow(t, b)
    inserti = &newb.tophash[0]
    insertk = add(newb, dataOffset)
    elem    = add(insertk, 8*KeySize)
}
```

Два условия роста (детали ниже). Если решили не расти, но места нет — создаём overflow-бакет. Это **аналог `rehash` + tombstone-логики** swiss, но проще: или растём, или удлиняем цепочку.

## 5. Запись ключа и метки [map_noswiss.go:713-725](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L713-L725)

```go
if t.IndirectKey() { /* аллоцируем ключ, кладём указатель */ }
if t.IndirectElem() { /* аллоцируем значение, кладём указатель */ }
typedmemmove(t.Key, insertk, key)   // копируем ключ
*inserti = top                      // помечаем ячейку занятой (tophash)
h.count++
```

## 6. Завершение [map_noswiss.go:727-735](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L727-L735)

```go
done:
if h.flags&hashWriting == 0 { fatal("concurrent map writes") }
h.flags &^= hashWriting             // снимаем флаг записи
if t.IndirectElem() { elem = *(*unsafe.Pointer)(elem) }
return elem                         // возвращаем адрес ячейки значения
```

## Карта приоритетов вставки

```
идём по бакету и его overflow-цепочке:
  ├─ нашли тот же ключ         → обновление (случай A), goto done
  ├─ нашли пустую ячейку       → запомнили как кандидата (inserti), идём дальше
  └─ дошли до конца цепочки    → ключа нет:
        ├─ перегруз / много overflow? → hashGrow + начать заново (goto again)
        ├─ был запомнен пустой слот?  → пишем туда
        └─ все полны                  → newoverflow + пишем в новый бакет
```

Сравни со swiss: там вместо `newoverflow` всегда `rehash`/`split` (open addressing не может «удлинить» — только перестроиться), а роль tombstone'ов в swiss здесь не нужна, потому что цепочки не требуют поддержания probe-инвариантов.

---

# Рост (`hashGrow`) + Эвакуация

## Два условия старта роста [map_noswiss.go:700](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L700)

1. **`overLoadFactor`** — превышен коэффициент загрузки (`13/16 ≈ 81%` от `8 * 2^B`) → рост **вдвое** (`B+1`).
2. **`tooManyOverflowBuckets`** — слишком много overflow-бакетов (примерно столько же, сколько обычных) → **рост того же размера** (`sameSizeGrow`), чтобы уплотнить данные и убрать «дырки» от удалений.

## `hashGrow` ничего не копирует [map_noswiss.go:1119](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L1119)

```go
bigger := uint8(1)
if !overLoadFactor(h.count+1, h.B) {
    bigger = 0
    h.flags |= sameSizeGrow         // рост без увеличения размера
}
oldbuckets := h.buckets
newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

h.B += bigger
h.oldbuckets = oldbuckets           // ← старый массив СОХРАНЯЕТСЯ
h.buckets = newbuckets              // новый пустой
h.nevacuate = 0
// данные НЕ переносятся здесь — только инкрементально
```

## Инкрементальная эвакуация [map_noswiss.go:1211](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L1211)

При каждой записи, пока идёт рост:

```go
func growWork(t, h, bucket) {
    evacuate(t, h, bucket & h.oldbucketmask())  // переносим бакет, который сейчас трогаем
    if h.growing() {
        evacuate(t, h, h.nevacuate)             // + ещё один, для прогресса
    }
}
```

`evacuate` раскидывает ключи старого бакета в новый массив. При росте вдвое каждый старый бакет делится на **два** по одному новому биту хеша:

```go
if hash & newbit != 0 { useY = 1 }   // → верхняя половина (Y)
else                   { useY = 0 }   // → нижняя половина (X)
dst := &xy[useY]                      // X или Y
// копируем пару в dst; старую ячейку метим evacuatedX / evacuatedY
```

Когда перенесён последний бакет (`nevacuate == #oldbuckets`), `oldbuckets = nil` — рост завершён [map_noswiss.go:1360-1362](vscode-webview://134g8j27pubsrs9od3idlmo4823tfa8108pm0gii6k1he6r1s7oq/src/runtime/map_noswiss.go#L1360-L1362).

---

# Главные отличия от swiss (для шпаргалки)

| Характеристика                 | Старая map (noswiss)                                                         | Новая map (swiss)                                 |
| ------------------------------ | ---------------------------------------------------------------------------- | ------------------------------------------------- |
| **Разрешение коллизий**        | overflow-цепочки (*chaining*, указатели)                                     | probing (*open addressing*)                       |
| **Контейнер**                  | bucket: `tophash[8]` + 8 ключей + 8 значений + overflow                      | group: control word + 8 slots                     |
| **Фильтр по хешу**             | `tophash`, цикл по 8 (последовательно)                                       | `H2` в control word, `matchH2` (SIMD/SWAR)        |
| **Биты хеша для адресации**    | младшие `B` бит → bucket                                                     | верхние биты → table                              |
| **Верхний уровень структуры**  | плоский массив `2^B` buckets                                                 | directory → tables → groups                       |
| **Удалённые ячейки**           | `emptyOne` / `emptyRest`, считаются пустыми                                  | tombstone (`deleted`), отдельная логика           |
| **Load Factor**                | `13/16 ≈ 81%`                                                                | `7/8 ≈ 87%`                                       |
| **Рост структуры**             | удвоение массива (`B+1`) или same-size grow; постепенная эвакуация bucket-ов | small → table → grow → split; перенос целой table |
| **Два массива во время роста** | ✅ Да (`buckets` + `oldbuckets`, lookup проверяет оба)                        | ❌ Нет, растёт независимая table                   |
| **Возврат из assign**          | указатель на value                                                           | указатель на value                                |
| **Защита от гонок**            | флаг `hashWriting`                                                           | флаг `writing` (XOR-toggle)                       |

Суть перехода: swiss убрал **указательные overflow-цепочки** (промахи кэша, мелкие аллокации) и **последовательную** проверку tophash, заменив их **плотными группами** и **параллельной** проверкой control word. Старая мапа проще по структуре (один массив бакетов), но платит за коллизии кэш-недружелюбными цепочками.
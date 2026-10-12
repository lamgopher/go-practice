# Занятие 09. Go + HTTP & JSON

Видео: https://www.youtube.com/watch?v=NsnDy0TAjj4 (1:48:21)
Расшифровка: `transcripts/09_NsnDy0TAjj4.txt`

## Что было на занятии

| Время | Тема |
|-------|------|
| 0:01 | `encoding/json`: `Marshal`, `MarshalIndent`, `Unmarshal` |
| 0:03 | Теги структур: переименование, `omitempty`, `-`, `"-,"`, `,string` для больших чисел; поля должны быть экспортируемыми |
| 0:09 | `json.Marshaler`: свой `MarshalJSON`; проверка реализации `var _ json.Marshaler = (*T)(nil)`; ловушка с получателем-указателем при передаче значения |
| 0:14 | Указатель или значение в получателе: стек, куча и GC |
| 0:16 | `json.Unmarshaler`: получатель обязан быть указателем |
| 0:18 | «Сломанный контракт»: поле `id` приходит то числом, то строкой — собственный тип с `UnmarshalJSON`. `go:embed` |
| 0:22 | Альтернативы: `jsoniter`, `easyjson`; когда они нужны |
| 0:31 | Инструменты для запросов: Postman, Insomnia, `.http`-файлы JetBrains, `curl` + `jq` |
| 0:36 | HTTP-клиент: `http.Get` и `DefaultClient` без таймаута; свой `http.Client{Timeout}`; `Transport`, `MaxIdleConns`, `MaxIdleConnsPerHost` (по умолчанию 2) |
| 0:41 | Контекст в запросе: `NewRequestWithContext`, побеждает меньший дедлайн; «открепление» контекста |
| 0:47 | `resp.Body.Close()` через `defer`; тело читается один раз |
| 0:50 | Заголовки (канонизация ключей), query-параметры через `url.Values`; POST с телом |
| 0:58 | HTTP-сервер: `ListenAndServe` (блокирует, всегда возвращает ошибку, `http.ErrServerClosed`), `HandleFunc`, `DefaultServeMux` |
| 1:03 | `http.Handler`, `ResponseWriter`, `*Request`, `r.Context()` |
| 1:07 | `http.NewServeMux` и его недостатки (методы, параметры пути, `/` как fallback); свой handler-обёртка с зависимостями; `gorilla/mux`, `chi` |
| 1:16 | Тесты: `httptest.NewRecorder`, `httptest.NewRequest`, вызов `ServeHTTP` без сети |
| 1:19 | Хендлер с зависимостью-интерфейсом `ProductProvider`; `mux.Vars`; маппинг ошибок на коды; `errors.Is`; ручной мок против генерируемого; mock и stub |
| 1:29 | `Content-Length`, `Content-Type`: зачем выставлять явно (`DetectContentType` дорогой и не знает JSON) |
| 1:33 | Graceful shutdown: сигнал → `server.Shutdown(ctx)` с таймаутом; что видит клиент без него |
| 1:41 | Два сервера (публичный и приватный с pprof и `/version`), `WaitGroup` при остановке |

## Вопросы для самопроверки

1. Что произойдёт при сериализации структуры с неэкспортируемыми полями?
2. Что делают теги `json:"name,omitempty"`, `json:"-"`, `json:"-,"`, `json:",string"`?
3. Зачем большие `int64` отдавать строкой?
4. Метод `MarshalJSON` объявлен на `*T`, а в `json.Marshal` передано значение `T`. Что произойдёт? Два способа исправить.
5. Почему у `UnmarshalJSON` получатель всегда указатель?
6. Как обработать поле, которое приходит то числом, то строкой?
7. Чем опасны `http.Get` и `http.DefaultClient` в продакшене?
8. У клиента таймаут 5 с, у контекста запроса 500 мс. Какой сработает? Можно ли «расширить» дедлайн дочерним контекстом?
9. Что означает `MaxIdleConnsPerHost` и когда значение по умолчанию становится проблемой?
10. Почему тело ответа нужно закрывать и почему его нельзя прочитать дважды?
11. Почему query-строку собирают через `url.Values`, а не `Sprintf`?
12. В каких двух случаях `ListenAndServe` возвращает ошибку? Какую из них считают нормой?
13. Из чего состоит интерфейс `http.Handler`? Что такое мультиплексор?
14. Какие недостатки стандартного `ServeMux` называл лектор? (Проверьте, какие из них исправлены в Go 1.22.)
15. Как протестировать хендлер без запуска сервера?
16. Чем mock отличается от stub? Аргументы лектора за ручные моки.
17. Почему `Content-Type: application/json` нужно ставить явно?
18. Что такое graceful shutdown? Что случится с запросом, выполняющимся 3 с, если таймаут завершения 1 с?

## Практика

### Задание 1. Теги и крайние случаи JSON

```go
type Item struct {
    ID      int64             `json:"id,string"`
    Name    string            `json:"name"`
    Price   float64           `json:"price,omitempty"`
    Tags    []string          `json:"tags,omitempty"`
    Meta    map[string]string `json:"meta"`
    secret  string
    Skip    string            `json:"-"`
    Dash    string            `json:"-,"`
    Created time.Time         `json:"created"`
    Parent  *Item             `json:"parent,omitempty"`
}
```

1. Предскажите JSON для `Item{}` и для полностью заполненной структуры, затем проверьте.
2. Разберите JSON с лишним полем, с отсутствующим полем, с `null` в `tags`, с `"id": 5` (числом). Что из этого ошибка?
3. Включите строгий режим: `json.NewDecoder(r).DisallowUnknownFields()`.
4. Вопрос слушателя с лекции остался без ответа: как записать и прочитать дату в нестандартном формате
   (`2006-01-02`)? Реализуйте тип `Date` с `MarshalJSON`/`UnmarshalJSON`.

### Задание 2. Кастомные (un)marshaler

1. Тип `FlexID int64`, принимающий и `3`, и `"3"` (пример лектора). Добавьте обработку `null` и мусора.
2. Тип `Status` (iota) должен сериализоваться строкой (`"active"`), а не числом.
3. Воспроизведите ловушку: `MarshalJSON` на указателе, значение в слайсе `[]T` и в поле структуры. Где метод вызывается?
4. Поле `payload` зависит от поля `type` (`"text"` → `TextPayload`, `"image"` → `ImagePayload`): разберите в два
   прохода через `json.RawMessage`.
5. Бенчмарк `encoding/json` против `jsoniter` на вашей структуре. Оправдана ли зависимость?

### Задание 3. HTTP-клиент

Напишите клиент к публичному API (например `https://httpbin.org`):

1. Собственный `http.Client` с таймаутом и `Transport` с настроенными `MaxIdleConns`, `MaxIdleConnsPerHost`, `IdleConnTimeout`.
2. `GET` с query-параметрами через `url.Values` и заголовками; `POST` с JSON-телом.
3. Все запросы — через `http.NewRequestWithContext`. Проверьте на `httpbin.org/delay/3`: таймаут клиента 5 с,
   контекста 1 с; затем наоборот.
4. Ответ не 2xx — ошибка с кодом и началом тела; тело всегда закрывается.
5. Эксперимент: прочитайте тело дважды. Напишите middleware-`RoundTripper`, логирующий тело ответа и возвращающий его
   вызывающему непрочитанным (`io.NopCloser(bytes.NewReader(...))`).
6. Два запроса параллельно через `errgroup.WithContext`; вариант «кто ответил первым, того и берём» с отменой второго.
7. Покройте клиент тестами через `httptest.NewServer`.

### Задание 4. HTTP-сервер

Сервис товаров:

```
GET    /products/{sku}   200 + JSON | 400 (sku не число или <= 0) | 404 | 500
POST   /products         201 + JSON | 400 (невалидный JSON)
GET    /version          версия Go, GOOS, GOARCH
```

1. Реализуйте дважды: на стандартном `http.ServeMux` (Go 1.22+: `mux.HandleFunc("GET /products/{sku}", ...)`) и на
   `chi` или `gorilla/mux`. Сравните с тем, что лектор говорил о стандартном роутере.
2. Хендлер зависит от интерфейса `ProductProvider { GetProduct(ctx, sku) (*Product, error) }`, объявленного рядом
   с хендлером; ошибка `ErrProductNotFound` проверяется через `errors.Is`.
3. Явный `Content-Type` — через middleware. Добавьте middleware логирования (метод, путь, код, длительность) и
   recovery.
4. Настройте `http.Server`: `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`.

### Задание 5. Тесты хендлеров

Табличный тест на `httptest.NewRecorder` + `httptest.NewRequest`:

| Случай | Ожидание |
|--------|----------|
| `GET /products/0` | 400 |
| `GET /products/abc` | 400 или 404 (зафиксируйте решение) |
| `GET /products/1`, провайдер не знает sku | 404 |
| `GET /products/1`, провайдер вернул другую ошибку | 500, внутренняя ошибка не утекает в ответ |
| `GET /products/1`, успех | 200, JSON с `sku: 1`, корректный `Content-Type` |
| `POST /products/1` | 405 |

Сделайте ручной мок провайдера (поле-функция или «известный sku»), затем сгенерируйте мок через `gomock` и
сравните читаемость.

### Задание 6. Graceful shutdown

1. Хендлер `/slow` спит 3 с. Скрипт в цикле шлёт запросы. Остановите сервер без `Shutdown` — зафиксируйте ошибку клиента.
2. Добавьте обработку `SIGINT`/`SIGTERM` и `server.Shutdown(ctx)` с таймаутом 1 с, затем 5 с. Сравните.
3. Поднимите второй, приватный сервер на другом порту (`/version`, `net/http/pprof`); останавливайте оба, дожидаясь
   через `WaitGroup` или `errgroup`.
4. Хендлер должен уважать отмену: если клиент отключился (`r.Context().Done()`), работа прекращается. Проверьте `curl` + Ctrl+C.

### Задание 7. Связь с проектом

Добавьте к сервису из задания 5 занятия 04 (или к API бота) HTTP-слой: хендлеры зависят от узких интерфейсов сервисов,
`main` собирает зависимости, graceful shutdown, тесты хендлеров без сети.

## Критерии готовности

- Ни одного `http.Get`/`DefaultClient` и ни одного запроса без контекста и таймаута.
- Каждый хендлер покрыт тестом через `httptest`; все ветки ошибок дают правильные коды.
- Сервер завершает начатые запросы при остановке — проверено руками.
- Умеете написать `MarshalJSON`/`UnmarshalJSON` и знаете ловушку с получателем.

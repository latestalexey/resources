# Создаём модель Bin и отвечаем на запросы
Не забываем загрузить код:
```bash
git checkout step-2
```
Размещать код внутри пакета main не очень правильно, так как, например [Google Application Engine](https://developers.google.com/appengine/docs/go/) создаёт свой пакет main, в котором уже подключаются ваши. Поэтому вынесем создание API в отдельный модуль, назовём его, например skimmer/api.go.

Теперь нам нужно создать сущность, в которой мы сможем хранить пойманные запросы, назовём её Bin, по аналогии с requestbin. Моделью у нас будет просто обычная структура данных Go.
> Порядок полей в структуре достаточно важен, но мы не будем задумываться об этом, но те кто хотят узнать как порядок влияет на размер структуры в памяти, могут почитать вот эти статьи — www.goinggo.net/2013/07/understanding-type-in-go.html и www.geeksforgeeks.org/structure-member-alignment-padding-and-data-packing.


Итак, наша модель Bin будет содержать поля с названием, количеством пойманных запросов, и датами создания и изменения. Каждое поле у нас так же описывается тэгом.
> Тэги это обычные строки, которые никак не влияют на программу в целом, но их можно прочитать используя пакет reflection во время работы программы (так называемая интроспекция), и исходя из этого изменять своё поведение (о том как работать тэгами через [reflection](http://golang.org/pkg/reflect/#StructTag)). В нашем примере, пакет json при кодировании/раскодировании учитывает значение тэга, примерно так:
> ```go
> package main
> 
> import (
>      "reflect"
>      "fmt"
> )
> 
> type Bin struct {
>      Name         string `json:"name"`
> }
> 
> func main() {
>      bin := Bin{}
>      bt := reflect.TypeOf(bin)
>      field := bt.Field(0)
>      fmt.Printf("Field's '%s' json name is '%s'", field.Name, field.Tag.Get("json"))
> }
> ```
> 
> Выведет
> Field's 'Name' json name is 'name' 
> 
> Пакет encoding/json поддерживает различные опции при формировании тэгов:
> ```go
> // Поле игнорируется
> Field int `json:"-"`
> 
> // В json структуре поле интерпретируется как myName
> Field int `json:"myName"`
> ```
> Вторым параметром может быть например, опция omitempty — если значение в json пропущено, то поле не заполняется. Так например, если поле будет ссылкой, мы сможем узнать, присутствует ли оно в json объекте, сравнив его с nil. Более подробно о json сериализации можно почитать в [документации](http://golang.org/pkg/encoding/json/)

Так же мы описываем вспомогательную функцию NewBin, в которой происходит инициализация значений объекта Bin (своего рода конструктор):
```go
type Bin struct {
        Name         string `json:"name"`
        Created      int64  `json:"created"`
        Updated      int64  `json:"updated"`
        RequestCount int    `json:"requestCount"`
}

func NewBin() *Bin {
        now := time.Now().Unix()
        bin := Bin{
                Created:      now,
                Updated:          now,
                Name:         rs.Generate(6),
        }
        return &bin
}
```

> Структуры в Go могут иницилизироваться двумя способами:
>
> 1) Обязательным перечислением всех полей по порядку:
> ```go
> Bin{rs.Generate(6), now, now, 0}
> ```
> 2) Указанием полей, для которых присваиваются значения:
> ```go
> Bin{
>          Created:      now,
>          Updated:      now,
>          Name:         rs.Generate(6),
>     }
> ```go
> Поля, которые не указаны, принимают значения по умолчанию. Например для целых чисел это будет 0, для строк — пустая строка "", для ссылок, каналов, массивов, слайсов и словарей — это будет nil. Подробнее в [документации](http://golang.org/ref/spec#The_zero_value). Главное помнить, что смешивать эти два типа инициализации нельзя.


Теперь более подробно про генерацию строк через объект rs. Он инициализирован следующим образом:
```go
var rs = NewRandomString("0123456789abcdefghijklmnopqrstuvwxyz")
```
Сам код находится в файле utils.go. В функцию мы передаём массив символов, из которых нужно генерировать строчку и создаём объект RandomString:
```go
type RandomString struct {
     pool string
     rg   *rand.Rand
}

func NewRandomString(pool string) *RandomString {
     return &RandomString{
          pool,
          rand.New(rand.NewSource(time.Now().Unix())),
     }
}

func (rs *RandomString) Generate(length int) (r string) {
     if length < 1 {
          return
     }
     b := make([]byte, length)
     for i, _ := range b {
          b[i] = rs.pool[rs.rg.Intn(len(rs.pool))]
     }
     r = string(b)
     return
}
```
Здесь мы используем пакет [math/rand](http://golang.org/pkg/math/rand), предоставляющий нам доступ к генерации случайных чисел. Самое главное, посеять генератор перед началом работы с ним, чтобы у нас не получилась одинаковая последовательность случайных чисел при каждом запуске.

В методе Generate мы создаём массив байтов, и каждый из байтов заполняем случайным символом из строки pool. Получившуюся в итоге строку возвращаем.

Перейдём, собственно, к описанию Api. Для начала нам нужно три метода для работы с объектами типа Bin, вывода списка объектов, создание и получение конкретного объекта.
Ранее я писал, что martini принимает в обработчик функцию с интерфейсом HandlerFunc, на самом деле, принимаемая функция в Martini описывается как interface{} — то есть это может быть абсолютно любая функция. Каким же образом в эту функцию вставляются аргументы? Делается это при помощи известного паттерна — [Dependency injection](http://en.wikipedia.org/wiki/Dependency_injection) (далее DI) при помощи небольшого пакета [inject](https://github.com/codegangsta/inject/) от автора martini. Не буду вдаваться в подробности относительно того, как это сделано, вы можете посмотреть в код самостоятельно, благо он не большой и там всё довольно просто. Но если двумя словами, то при помощи уже упомянутого пакета reflect, получаются типы аргументов функции и после этого подставляются нужные объекты этого типа. Например когда inject видит тип *http.Request, он подставляет объект req *http.Request в этот параметр.
Мы можем сами добавлять нужные объекты для рефлексии через методы объекта Map и MapTo глобально, либо через объект контекста запроса martini.Context для каждого запроса отдельно.

Объявим временные переменные history и bins, первый будет содержать историю созданных нами объектов Bin, а второй будет некой куцей версией хранилища объектов Bin.
Теперь рассмотрим созданные методы.

### Создание объекта Bin
```go
api.Post("/api/v1/bins/", func(r render.Render){
       bin := NewBin()
       bins[bin.Name] = bin
       history = append(history, bin.Name)
       r.JSON(http.StatusCreated, bin)
  })
```
### Получение списка объектов Bin
```go
api.Get("/api/v1/bins/", func(r render.Render){
       filteredBins := []*Bin{}
       for _, name := range(history) {
            if bin, ok := bins[name]; ok {
                 filteredBins = append(filteredBins, bin)
            }
       }
       r.JSON(http.StatusOK, filteredBins)
  })
```
Получение конкретного экземпляра
```go
api.Get("/api/v1/bins/:bin", func(r render.Render, params martini.Params){
       if bin, ok := bins[params["bin"]]; ok{
            r.JSON(http.StatusOK, bin)
       } else {
            r.Error(http.StatusNotFound)
       }
  })
```
Метод позволяющий получить объект Bin по его имени, в нём мы используем объект martini.Params (по сути просто map[string]string), через который можем доступиться к разобранным параметрам адреса.
> В языке Go мы можем обратиться к элементу словаря двумя способами:
> Запросив значение ключа a := m[key], в этом случае вернётся либо значение ключа в словаре, если оно есть, либо дефолтное значение инициализации типа значения. Таким образом, например для чисел, сложно понять, содержит ли ключ 0 или просто значения этого ключа не существует. Поэтому в го предусмотрен второй вариант.
> В этом способе, запросив по ключу и получить его значение первым параметром и индикатор существования этого ключа вторым параметром — a, ok := m[key]


Поэкспериментируем с нашим приложением. Для начала запустим его:
```bash
go run ./src/main.go
```
Добавим новый объект Bin:
```bash
> curl -i -X POST "127.0.0.1:3000/api/v1/bins/"
HTTP/1.1 201 Created
Content-Type: application/json; charset=UTF-8
Date: Mon, 03 Mar 2014 04:10:38 GMT
Content-Length: 76

{"name":"7xpogf","created":1393819838,"updated":1393819838,"requestCount":0}
```
Получим список доступных нам Bin объектов:
```bash
> curl -i "127.0.0.1:3000/api/v1/bins/"
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Date: Mon, 03 Mar 2014 04:11:18 GMT
Content-Length: 78

[{"name":"7xpogf","created":1393819838,"updated":1393819838,"requestCount":0}]
```
Запросим конкретный объект Bin, взяв значение name из предыдущего запроса:
```bash
curl -i "127.0.0.1:3000/api/v1/bins/7xpogf"
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Date: Mon, 03 Mar 2014 04:12:13 GMT
Content-Length: 76

{"name":"7xpogf","created":1393819838,"updated":1393819838,"requestCount":0}
```
Отлично, теперь мы научились создавать модели и отвечать на запросы, кажется теперь нас ничего не удержит от того, чтобы доделать всё остальное.

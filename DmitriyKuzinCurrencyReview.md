деплой: http://31.40.61.92:8080/
гит: https://github.com/Exelent26/Currencies_Exchange
[Дмитрий Кузин]



## ХОРОШО

1. Весь функционал реализован по ТЗ
2. Обработаны крайние случаи при добавлении/измении обмена валют
3. Реализована валидация при создании валиюты

## Недостатки реализации

1. Нет отображения ошибок на фронте

## ЗАМЕЧАНИЯ/Возможности для улучшения

**1. Пакет Filter**

- Используешь фильтр только для обработки ответа сервера. 
- Стоит рассмотреть использование фильтра для обработки ошибок
- Пример :

```java
protected void doFilter(HttpServletRequest req, HttpServletResponse res, FilterChain chain) throws ServletException, IOException {
    try {
        super.doFilter(req, res, chain);
    }
    catch (DatabaseOperationException e) {
        writeErrorResponse(res, SC_INTERNAL_SERVER_ERROR, e);
    }
    }
```

**2. Пакет Dao**  

Хорошо:
- Используешь интерфейс для определения необходимых методов
- Вынес строки запросов в константы
- реализовал паттерн синглтон
- Молодец что сам реализовал конекшн пул но на будущее советую почитать про HikariCP

Замечания:
- Не понял почему у тебя этот метод публичный. Где ты его еще используешь?
```java
 public static Currency buildCurrency(ResultSet rs) throws SQLException {
    return new Currency(rs.getInt("id"),
            rs.getString("code"),
            rs.getString("full_name"),
            rs.getString("sign"));
}
```
- Не совсем понимаю почему в этом методе ты отдельно передаешь Connection но при этом используешь все равно тот который импортировал и при вызове метода не передаешь Connection вторым аргументом
```java
  public Optional<Currency> findByCode(String code, Connection connection) {}
```
- Мне не нравится что ты проводишь валидацию на уровне ДАО тем более в самих методах отвечающих за взаимодействие с БД лучше проводить валидацию на уровне сервиса и создать для этого отдельный класс как ты и сделал (DataValidator.class)
Пример:
```java
      public ExchangeRate save(String baseCode, String targetCode, BigDecimal rate) {
    if (baseCode == null || rate == null || targetCode == null) {
        throw new DaoException("Invalid input: baseCode or targetCode  or rate is null", DaoException.ErrorCode.INVALID_INPUT);
    }
}
```

- Почему ты отлавливаешь экспешн только в самом методе сохранения курса валюты лучше сделать проверку в методе поиска валюты что например если список пуст то пробрасываем исключения что валюта не нацдена

Пример:
```java
          public ExchangeRate save(String baseCode, String targetCode, BigDecimal rate) {
                CurrencyDao currencyDao = CurrencyDao.getInstance();
                Currency base = currencyDao.findByCode(baseCode, connection).orElseThrow(() -> new DaoException(
                        "Base currency not found: " + baseCode,
                        DaoException.ErrorCode.CURRENCY_NOT_FOUND));
}
```

- Старайся выносить любые непонятный числа в константы класса или в утильный класс но чтобы они не были просто числами потому что например не знающий человек не поймет что такое 19

```java
         if (e.getErrorCode() == 19) {
        throw new DaoException("Currencies pair rate already exists", DaoException.ErrorCode.DUPLICATE_EXCHANGE_RATE);
                }
```
Задание со звездочкой:
- Ели внимательно посмотришь на свой код то можешь заметить что у тебя постоянно повторяются участки кода с подключением к БД и созданием prepare statement можешь подумать или посмотреть в других реализациях как это можно вынести вынести в какие то утильные методы интерфейса


**3. Пакеты DTO и Entity**  
Варианты для улучшения:
- Как можешь заметить у тебя очень много шаблонного кода по типу геттеров сеттеров и конструкторов почитай про Lombok это библиотека которая заменяет все эти методы аннотациями которые ты прописываешь над классом 

**4. Пакет Exception**   
Замечания
- Если уже решил использовать ENUM для константных наименований ошибок то лучше вынести всех их в отдельный(ые) класс(ы) 

**5. Пакет Mapper**  
Молодец что сам попробовал реализовать преобразование Dto  в сущности и наоборот но на будущее лучше использовать для этого библиотеки например ModelMapper

**Пакет Service**

Замечания:
- Я бы изменил обработку ошибок почему бы тебе не пробрасывать исключение сразу в методах валидации тогда тебе не нужно будет проверять отдельно методы валидации в сервисах методы. Как итог методы сервиса будут чище и понятнее
- У тебя получились супер методы которые выглядят очень большими и плохо читаемыми подумай как их можно сократить или разделить на несколько методов. Они такими вышли во многом из за того что ты делаешь в них валидацию
```java
    public ExchangeRate createNewExchangeRate(String baseCurrencyCode, String targetCurrencyCode, String rateString) {}
    public ExchangeDto performExchange(String baseCurrency, String targetCurrency, String amountString) {}
public ExchangeRate updateExchangeRate(String baseCurrencyCode, String targetCurrencyCode, String rateString) {}
```
- Зачем тебе для сохранения отдельный метод в котором ты по факту просто вызываешь метод save у DAO?
```java
      public ExchangeRate save(ExchangeRate exchangeRate) {
    try {
        return exchangeRateDao.save(exchangeRate);
    } catch (DaoException e) {
        throw new ServiceException("Can't save exchangeRate", e, ServiceException.ErrorCode.DAO_ERROR);
    }
}
```

- Аналогичная проблема с предыдущим замечанием зачем тебе отдельный метод который просто вызывает конструктор?
```java
         public ExchangeRate buildExchangeRate(Currency baseCurrency, Currency targetCurrency, BigDecimal rate) {
    return new ExchangeRate(baseCurrency, targetCurrency, rate);
}
```
- Есть неиспользуемые методы которые можно убрать
Пример:
```java
      public ExchangeRate updateExchangeRate(Integer exchangeRateId, BigDecimal rate) {
    try {
        return exchangeRateDao.updateExchangeRate(exchangeRateId, rate.setScale(2, RoundingMode.HALF_UP));
    } catch (DaoException e) {
        throw new ServiceException("Can't update exchangeRate", e, ServiceException.ErrorCode.DAO_ERROR);
    }
}
```

**7. Пакет Servlet**

Замечания:
- Как уже упоминал ранее стоит подумать над обработкой исключений через фильтры сейчас сервлеты выглядят достаточно перегруженными в том числе из за обработки ошибок
- Основные задачи сервлета это: Прием параметров, возможна валидация этих параметров и отправка ответа
- Лучше не использвать ключевое слово var где попало подробнее можно почитать например здесь https://habr.com/ru/articles/438206/
Пример в твоем коде:
```java
      @WebServlet("/exchangeRates")
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)  {
        try {
            var exchangeRates = exchangeRateService.getExchangeRates();
```
- Странно зачем ты здесь отдельно проверяешь на наличие в бд если при сохранении уже имеющегося обменного курса выдаст unique constraint ошибку 
```java
      @WebServlet("/exchangeRates")
    @Override
protected void doPost(HttpServletRequest request, HttpServletResponse response)  {
    Optional<ExchangeRateDto> exchangeRateByCode = exchangeRateService.getExchangeRateByCode(baseCurrencyCode, targetCurrencyCode);

    if (exchangeRateByCode.isPresent()) {
        response.setStatus(HttpServletResponse.SC_CONFLICT);
        return;
    }

    ExchangeRate exchangeRate = exchangeRateService.createNewExchangeRate(baseCurrencyCode, targetCurrencyCode, rateString);
    response.setStatus(HttpServletResponse.SC_CREATED);
```


## ВЫВОД

В целом хорошая реализация главное что все работает соглласно ТЗ.
Стоит посмотреть ролики Сергея про MVC и четко разраничить обязанности между слоями
Обрати внимание на обработку ошибок. Одни и те же ошибки ты пытаешься отловить на разных слоях скорее всего не очень понимаешь зоны ответственности. Если у тебя например ошибка в DAO или Валидации то отлавливай её только там ОДИН раз 
Обрати внимание на библиотеки и модуль которые подсветил в ревью (Hikari, lombok, modelmapper)

#ревью #обмен_валют
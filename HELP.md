# Теорія написання DDD, і не тільки

# 🎯 Використання Value Objects у DDD

У доменно-орієнтованому проєктуванні (DDD) часто застосовується практика створення окремих класів або record-ів для полів у доменних моделях. Це називається Value Object підхід.

🔍 Чому це важливо?

Валідація на рівні моделі

Замість того, щоб перевіряти правильність значення в кожному місці коду, ми гарантуємо коректність ще на етапі створення об’єкта.

Наприклад, ProductName завжди не може бути порожнім, а Price має бути додатним.

Інкапсуляція бізнес-логіки

Логіка, що стосується конкретного атрибуту, зберігається разом із ним.

Наприклад, Price може мати метод applyDiscount(Percentage) замість винесення цієї логіки у сервіс.

Семантична зрозумілість

Код стає більш виразним: замість простого String або BigDecimal ми бачимо ProductName, ProductDescription, Price.

Це зменшує плутанину та підвищує читабельність.

Захист від помилок

Неможливо випадково передати не той параметр: наприклад, ProductName не можна переплутати з CategoryName, якщо вони різні типи.

Імм’ютабельність

Більшість Value Objects роблять незмінними (record у Java або final-поля).

Це підвищує надійність та полегшує роботу з багатопоточною обробкою.

📦 Приклад:
public record ProductName(String value) {
public ProductName {
if (value == null || value.isBlank()) {
throw new IllegalArgumentException("Product name cannot be empty");
}
}
}

public record Price(BigDecimal value) {
public Price {
if (value == null || value.compareTo(BigDecimal.ZERO) <= 0) {
throw new IllegalArgumentException("Price must be greater than zero");
}
}

    public Price applyDiscount(BigDecimal percentage) {
        return new Price(value.subtract(value.multiply(percentage)));
    }
}


Тепер у доменній моделі Product ми працюємо не з "сирими" типами, а з перевіреними обгортками:

public class Product {
private final ProductName name;
private final Price price;
private final ProductDescription description;
// ...
}


✅ Такий підхід допомагає зробити доменну модель більш захищеною, виразною і ближчою до бізнес-термінів, а не до технічних примітивів.

# Репозиторії в DDD

# Репозиторії у DDD – розширена шпаргалка

1. **Репозиторії оперують тільки доменними об'єктами (Aggregate Roots).**
    - Вони не знають про DTO, контролери або зовнішні сервіси.

2. **Ніяких DTO тут бути не повинно.**
    - DTO належать application або web шару.

3. **Стандартний набір методів:**
    - `save(AggregateRoot aggregate)` – зберегти або оновити агрегат
    - `findById(Id id)` – знайти агрегат за ідентифікатором
    - `findAll()` – отримати всі агрегати
    - `delete(AggregateRoot aggregate)` – видалити агрегат
    - `existsById(Id id)` – перевірити наявність агрегату
    - `count()` – підрахувати кількість агрегатів
    - `findBySomeField(FieldType field)` – пошук за іншими критеріями (опційно)
    - `findAllBySpecification(Specification spec)` – для складних запитів (якщо використовуєте Spring Data JPA)
    - `flush()` – примусове збереження змін у БД (якщо потрібно)

4. **Оновлення Aggregate Root:**
    - Завантажуємо об’єкт з репозиторію:
      ```java
      Product product = productRepository.findById(id)
          .orElseThrow(() -> new RuntimeException("Product not found"));
      ```
    - Викликаємо методи бізнес-логіки на Aggregate Root:
      ```java
      product.setPrice(new Price(100));
      product.addCategory(category);
      ```
    - Повторно зберігаємо об’єкт у репозиторій:
      ```java
      productRepository.save(product);
      ```

5. **Репозиторій не містить бізнес-логіки** – вона знаходиться в Aggregate Root або Domain Service.



# Принцип розташування сервісів
## 🔹 1. Application Services (Application Layer)

**Призначення:**

* Координують **бізнес-процеси**.
* Викликають методи **доменної моделі** (Entities, Aggregates).
* Використовують **репозиторії** для збереження/отримання агрегатів.
* Не містять бізнес-логіку агрегатів (тільки orchestration).

**Розташування в проекті:**

```
com.edu.web.spacecatsmarket.catalog.application
    └─ ProductService.java
```

**Приклад поведінки:**

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public void updateProduct(ProductId id, UpdateProductDTO dto) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Product not found"));

        product.setName(dto.getName());
        product.setPrice(dto.getPrice());

        productRepository.save(product);
    }
}
```

---

## 🔹 2. Domain Services (Domain Layer)

**Призначення:**

* Використовуються, коли бізнес-логіка **не належить конкретному агрегату**, але стосується декількох агрегатів або сутностей.
* Вони **є частиною домену**, містять справжню бізнес-логіку.

**Приклад:**

* Сервіс, який перевіряє, чи можна об’єднати товари з різних категорій у замовленні:

```java
package com.edu.web.spacecatsmarket.catalog.domain;

public class PricingService {

    public Price calculateDiscountedPrice(Product product, Customer customer) {
        // складна бізнес-логіка
    }
}
```

* Може бути **ін’єктований у Application Service** для використання.

---

## 🔹 3. Infrastructure Services (Infrastructure Layer)

**Призначення:**

* Технічні сервіси: email, SMS, платіжні шлюзи, зовнішні API.
* Можуть мати @Service, але **не містять бізнес-логіку домену**.

**Приклад:**

```java
@Service
public class EmailNotificationService {

    public void sendProductUpdateNotification(User user, Product product) {
        // Виклик зовнішнього SMTP-сервера
    }
}
```

---

## 🔹 4. Основні правила

| Тип сервісу                | Де лежить        | Містить                                | Приклад                  |
| -------------------------- | ---------------- | -------------------------------------- | ------------------------ |
| **Application Service**    | `application`    | orchestration: репозиторії + домен     | ProductService           |
| **Domain Service**         | `domain`         | бізнес-логіку, що не належить агрегату | PricingService           |
| **Infrastructure Service** | `infrastructure` | технічні інтеграції                    | EmailNotificationService |

---

# ⚠️ Обробка помилок за стандартом RFC 9457 (Problem Details for HTTP APIs)

## 🔄 Відмінності між RFC 9457 та RFC 7807
Новий стандарт **RFC 9457** замінив попередній **RFC 7807** та уточнив правила опису помилок у REST API:

- **Розширення (extension members)**  
  Більш чітко описано, що клієнти повинні **ігнорувати несподівані поля** у відповіді помилки.  
  Це дозволяє розробникам додавати власні атрибути (`errors`, `traceId`, `code` тощо), не ламаючи клієнтів.

- **Реєстри типів проблем (Problem Type Registries)**  
  Введено можливість стандартизувати типи помилок через URI або зареєстровані ідентифікатори.  
  Це дозволяє API чітко визначати **категорії проблем** і робити їх машино-зрозумілими.

- **Уточнення правил**  
  Додані ясніші рекомендації щодо:
    - використання полів `status`, `title`, `detail`
    - коректної декларації `instance` (URI, що вказує на конкретний випадок помилки)
    - правил іменування типів проблем.

---

## ⚙ Використання у Spring (Spring Framework / Spring Boot)

Spring має вбудовану підтримку **Problem Details / RFC 9457** у Spring MVC:

- **`ProblemDetail`**  
  Стандартний клас-контейнер для полів: `type`, `status`, `title`, `detail`, `instance`.

- **`ErrorResponse`**  
  Інтерфейс, який дозволяє виняткам інкапсулювати інформацію про HTTP-статус та тіло помилки у форматі RFC 9457.

- **Глобальна обробка через `@ControllerAdvice` або `ResponseEntityExceptionHandler`**  
  У цих класах можна повертати `ProblemDetail` або `ErrorResponse`, і Spring автоматично відправить відповідь із MIME-типом  
  **`application/problem+json`**.

- **Підтримка залежить від версії Spring Boot**  
  У нових версіях Spring (6.x / Boot 3.x) підтримка RFC 9457 є повною, у старіших — частковою.

---

✅ Рекомендація:  
Для уніфікованої обробки помилок у REST API варто завжди повертати відповіді у форматі **Problem Details**,  
а додаткові дані (наприклад, список валідаційних помилок) додавати як розширені поля.




# Домашнее задание: Репликация и масштабирование. Часть 2

---

## Задание 1

**Условие:**  
Опишите основные преимущества использования масштабирования методами:

- активный master-сервер и пассивный репликационный slave-сервер;  
- master-сервер и несколько slave-серверов.  

**Ответ:**

- **Активный master и пассивный slave**  
  - Повышается отказоустойчивость: при сбое master можно переключиться на slave.  
  - Slave можно использовать для резервного копирования, чтобы не нагружать master.  
  - Простая архитектура, подходит для систем со средней нагрузкой.  

- **Master и несколько slave-серверов**  
  - Масштабируемость по чтению: запросы SELECT можно распределять между несколькими slave.  
  - Снижение нагрузки на master, который обрабатывает только записи.  
  - Возможность геораспределённого размещения slave ближе к пользователям.  
  - Гибкость — разные slave могут использоваться для аналитики и отчётности.  

---

## Задание 2

**Условие:**  
Разработайте план для выполнения горизонтального и вертикального шардинга базы данных.  
База данных состоит из трёх таблиц:  
- пользователи,  
- книги,  
- магазины (столбцы произвольные).  

---

### 1) Вертикальный шардинг (разделение по доменам)

- **Auth/User DB** — хранит пользователей (`users`, `user_profiles`, `auth_sessions`).  
- **Books DB** — хранит книги (`books`, `authors`, `genres`, `book_inventory`).  
- **Stores DB** — хранит магазины (`stores`, `store_inventory`, `store_orders`).  

---

### 2) Горизонтальный шардинг (масштабирование внутри домена)

- **Users:** шард по `user_id % N`.  
- **Books:** шард по `book_id` или `isbn_hash % N`.  
- **Stores:** региональный шард по `region_code`.  

Каждый шард: `Primary (RW) + Replica (RO)`.

---

### 3) Роутинг и метаданные

- Routing: ProxySQL / MySQL Router.  
- Таблица `shard_map`: где хранится карта соответствия `ключ → шард`.

---

### 4) Режимы работы серверов

- Primary — принимает запись и читает.  
- Replica — только чтение, балансировка запросов.  
- Failover — автоматический перевод Replica в Primary.  

---

### 5) Пример таблиц

```sql
-- Users DB
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,
  email   VARCHAR(255) UNIQUE,
  name    VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Books DB
CREATE TABLE books (
  book_id   BIGINT PRIMARY KEY,
  isbn      VARCHAR(20) UNIQUE,
  title     VARCHAR(255),
  author_id BIGINT,
  published_at DATE
);

-- Stores DB
CREATE TABLE stores (
  store_id    BIGINT PRIMARY KEY,
  region_code VARCHAR(8),
  name        VARCHAR(255),
  address     VARCHAR(255)
);


⸻

6) Блок-схема

flowchart LR
  subgraph Clients[Сервисы]
    A[Auth Service]
    B[Books Service]
    C[Stores Service]
  end

  R[Router/Proxy\n(RW->Primary, RO->Replica)]

  A --> R
  B --> R
  C --> R

  subgraph Users["Users DB"]
    U0P[(Shard U0 Primary)] --> U0R1[(Replica)]
    U1P[(Shard U1 Primary)] --> U1R1[(Replica)]
  end

  subgraph Books["Books DB"]
    B0P[(Shard B0 Primary)] --> B0R1[(Replica)]
    B1P[(Shard B1 Primary)] --> B1R1[(Replica)]
  end

  subgraph Stores["Stores DB"]
    S1[(Region R1 Primary)] --> S1R[(Replica)]
    S2[(Region R2 Primary)] --> S2R[(Replica)]
  end

  R --> U0P
  R --> U1P
  R --> B0P
  R --> B1P
  R --> S1
  R --> S2


⸻


# Source DB Project — Детальні пояснення

У цьому документі наведено весь код і детальні пояснення до кожного файлу та кожного ключового кроку для роботи з базою `source_db` за допомогою Java DAO.

---

## 1. SQL: `create_source_db.sql`

```sql
-- Створення бази даних, якщо вона відсутня
CREATE DATABASE IF NOT EXISTS source_db;
-- Перемикаємося на щойно створену схему
USE source_db;

-- Створюємо таблицю source:
CREATE TABLE IF NOT EXISTS source (
    id INT AUTO_INCREMENT PRIMARY KEY,   -- унікальний ідентифікатор запису
    name VARCHAR(255) NOT NULL,          -- назва джерела (обов'язкове поле)
    url VARCHAR(512),                    -- URL джерела (необов'язкове поле)
    description VARCHAR(512)             -- опис джерела (необов'язкове поле)
);
```

**Пояснення рядок за рядком**  
- `CREATE DATABASE IF NOT EXISTS source_db;` — створює схему з ім’ям `source_db`, якщо вона ще не існує, запобігаючи помилці при повторному запуску.  
- `USE source_db;` — вказує MySQL виконувати подальші команди в контексті цієї бази.  
- `CREATE TABLE IF NOT EXISTS source (...);` — створює таблицю `source` лише якщо вона відсутня.  
  - `id INT AUTO_INCREMENT PRIMARY KEY` — поле `id` автоматично збільшується при вставці нового рядка; визначається як первинний ключ.  
  - `name VARCHAR(255) NOT NULL` — зберігає назву джерела; `NOT NULL` забезпечує, що значення не може бути пустим.  
  - `url VARCHAR(512)` — може містити адресу веб-сторінки джерела.  
  - `description VARCHAR(512)` — вільний текстовий опис джерела.

---

## 2. JDBC-підключення: `DatabaseConnection.java`

```java
package com.example.util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

/**
 * Клас для керування JDBC-з’єднанням з базою source_db.
 * Використовується патерн Singleton для повторного використання одного з’єднання.
 */
public class DatabaseConnection {
    private static final String URL =
        "jdbc:mysql://localhost:3306/source_db?useSSL=false&serverTimezone=UTC";
    private static final String USER = "root";                    // ім'я користувача БД
    private static final String PASSWORD = "your_password_here";   // пароль до БД

    private static Connection connection;

    private DatabaseConnection() { }

    /**
     * Повертає активне з’єднання або створює нове, якщо його ще не було.
     * @return Connection об’єкт для виконання SQL-запитів.
     * @throws SQLException у разі проблем з підключенням.
     */
    public static Connection getConnection() throws SQLException {
        // Перевіряємо наявність і стан попереднього з’єднання
        if (connection == null || connection.isClosed()) {
            connection = DriverManager.getConnection(URL, USER, PASSWORD);
        }
        return connection;
    }
}
```

**Пояснення**  
- `DriverManager.getConnection(URL, USER, PASSWORD)` відкриває TCP/IP з’єднання з MySQL.  
- Статична змінна `connection` зберігає єдине з’єднання.  
- Повторне використання одного з’єднання підвищує продуктивність і зменшує навантаження.

---

## 3. Модель: `Source.java`

```java
package com.example.model;

/**
 * POJO-клас, що відповідає рядку таблиці source.
 */
public class Source {
    private int id;               // PK, відповідає AUTO_INCREMENT
    private String name;          // назва джерела
    private String url;           // URL джерела
    private String description;   // опис джерела

    // Конструктор без параметрів потрібен для фреймворків і десеріялізації
    public Source() { }

    // Використовується при вставці нового запису (без id)
    public Source(String name, String url, String description) {
        this.name = name;
        this.url = url;
        this.description = description;
    }

    // Використовується при читанні з бази з наявним id
    public Source(int id, String name, String url, String description) {
        this(name, url, description);
        this.id = id;
    }

    // Геттери і сеттери для доступу до полів
    public int getId() { return id; }
    public void setId(int id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    @Override
    public String toString() {
        return "Source{" +
               "id=" + id +
               ", name='" + name + ''' +
               ", url='" + url + ''' +
               ", description='" + description + ''' +
               '}';
    }
}
```

**Пояснення**  
- POJO забезпечує чітку відповідність полям таблиці.  
- Конструктори та геттери/сеттери дозволяють легко створювати і модифікувати об’єкт.

---

## 4. DAO-інтерфейс: `SourceDAO.java`

```java
package com.example.dao;

import com.example.model.Source;
import java.sql.SQLException;
import java.util.List;

/**
 * Інтерфейс задає CRUD-операції для роботи з таблицею source.
 */
public interface SourceDAO {
    void addSource(Source source) throws SQLException;
    Source getSourceById(int id) throws SQLException;
    List<Source> getAllSources() throws SQLException;
    void updateSource(Source source) throws SQLException;
    void deleteSource(int id) throws SQLException;
}
```

**Пояснення**  
- Методи відповідають базовим операціям: створення (Create), читання (Read), оновлення (Update), видалення (Delete).  
- Виняток `SQLException` дає клієнту можливість обробити помилки на рівні БД.

---

## 5. Реалізація DAO: `SourceDAOImpl.java`

```java
package com.example.dao;

import com.example.model.Source;
import com.example.util.DatabaseConnection;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

/**
 * Реалізація джерел (Source) через JDBC.
 */
public class SourceDAOImpl implements SourceDAO {
    private final Connection conn;

    public SourceDAOImpl() throws SQLException {
        // Отримуємо з’єднання
        this.conn = DatabaseConnection.getConnection();
    }

    @Override
    public void addSource(Source source) throws SQLException {
        String sql = "INSERT INTO source (name, url, description) VALUES (?, ?, ?)";
        try (PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, source.getName());
            ps.setString(2, source.getUrl());
            ps.setString(3, source.getDescription());
            ps.executeUpdate();
            try (ResultSet rs = ps.getGeneratedKeys()) {
                if (rs.next()) {
                    source.setId(rs.getInt(1));  // витягуємо згенерований id
                }
            }
        }
    }

    @Override
    public Source getSourceById(int id) throws SQLException {
        String sql = "SELECT * FROM source WHERE id = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return new Source(
                        rs.getInt("id"),
                        rs.getString("name"),
                        rs.getString("url"),
                        rs.getString("description")
                    );
                }
            }
        }
        return null; // повертається null якщо запис не знайдено
    }

    @Override
    public List<Source> getAllSources() throws SQLException {
        List<Source> list = new ArrayList<>();
        String sql = "SELECT * FROM source";
        try (Statement st = conn.createStatement();
             ResultSet rs = st.executeQuery(sql)) {
            while (rs.next()) {
                list.add(new Source(
                    rs.getInt("id"),
                    rs.getString("name"),
                    rs.getString("url"),
                    rs.getString("description")
                ));
            }
        }
        return list;
    }

    @Override
    public void updateSource(Source source) throws SQLException {
        String sql = "UPDATE source SET name=?, url=?, description=? WHERE id=?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, source.getName());
            ps.setString(2, source.getUrl());
            ps.setString(3, source.getDescription());
            ps.setInt(4, source.getId());
            ps.executeUpdate();
        }
    }

    @Override
    public void deleteSource(int id) throws SQLException {
        String sql = "DELETE FROM source WHERE id=?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            ps.executeUpdate();
        }
    }
}
```

**Пояснення методів**  
- `addSource(...)` — вставка нового запису з подальшим отриманням auto-generated ключа.  
- `getSourceById(...)` — запит із параметром `id`, повертає один об’єкт або `null`.  
- `getAllSources()` — отримує список всіх джерел.  
- `updateSource(...)` — оновлює поля запису за `id`.  
- `deleteSource(...)` — видаляє запис за `id`.

---

## 6. Демонстрація: `Main.java`

```java
package com.example;

import com.example.dao.SourceDAO;
import com.example.dao.SourceDAOImpl;
import com.example.model.Source;
import java.sql.SQLException;
import java.util.List;

/**
 * Приклад використання SourceDAO.
 */
public class Main {
    public static void main(String[] args) {
        try {
            SourceDAO sourceDao = new SourceDAOImpl();

            // 1) CREATE — додаємо новий запис
            Source src = new Source("Example Source", "https://example.com", "Example description");
            sourceDao.addSource(src);
            System.out.println("Added: " + src);

            // 2) READ — отримуємо всі записи
            List<Source> all = sourceDao.getAllSources();
            System.out.println("All sources:");
            all.forEach(System.out::println);

            // 3) UPDATE — змінюємо опис
            src.setDescription("Updated description");
            sourceDao.updateSource(src);
            System.out.println("After update: " + sourceDao.getSourceById(src.getId()));

            // 4) DELETE — видаляємо запис
            sourceDao.deleteSource(src.getId());
            System.out.println("After delete:");
            sourceDao.getAllSources().forEach(System.out::println);

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

**Пояснення сценарію**  
1. **CREATE**: створення нового об’єкта і збереження.  
2. **READ ALL**: друк поточного стану таблиці.  
3. **UPDATE**: модифікація полів існуючого запису.  
4. **DELETE**: видалення запису і фінальний огляд таблиці.  

---

## Результати

**JAVA**

![Diagram](j.png)

**MySQL Shell**

![Diagram](Shell.png)

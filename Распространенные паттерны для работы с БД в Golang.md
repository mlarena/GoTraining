# Распространенные паттерны для работы с БД в Golang

В Go существует несколько популярных паттернов для работы с базами данных. Вот основные из них:

## 1. Repository Pattern

Паттерн репозитория абстрагирует доступ к данным, предоставляя коллекцию-подобный интерфейс.

```go
type UserRepository interface {
    Get(id int) (*User, error)
    Create(user *User) error
    Update(user *User) error
    Delete(id int) error
}

type userRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Get(id int) (*User, error) {
    // реализация запроса к БД
}
```

## 2. Data Mapper

Отображает данные между объектами и БД, сохраняя их независимыми друг от друга.

```go
type UserMapper struct {
    db *sql.DB
}

func (m *UserMapper) FindByID(id int) (*User, error) {
    row := m.db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id)
    user := &User{}
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    return user, err
}
```

## 3. Active Record

Объект содержит как данные, так и методы для работы с ними.

```go
type User struct {
    ID    int
    Name  string
    Email string
    db    *sql.DB
}

func (u *User) Save() error {
    if u.ID == 0 {
        // INSERT
        result, err := u.db.Exec("INSERT INTO users...")
        id, _ := result.LastInsertId()
        u.ID = int(id)
        return err
    }
    // UPDATE
    _, err := u.db.Exec("UPDATE users SET... WHERE id = ?", u.ID)
    return err
}
```

## 4. Unit of Work

Группирует несколько операций в одну транзакцию.

```go
type UnitOfWork struct {
    db *sql.DB
    tx *sql.Tx
}

func (uow *UnitOfWork) Begin() error {
    tx, err := uow.db.Begin()
    uow.tx = tx
    return err
}

func (uow *UnitOfWork) Commit() error {
    return uow.tx.Commit()
}

func (uow *UnitOfWork) Rollback() error {
    return uow.tx.Rollback()
}
```

## 5. DAO (Data Access Object)

Предоставляет абстрактный интерфейс к конкретному источнику данных.

```go
type UserDAO interface {
    Insert(user User) error
    Get(id int) (User, error)
    GetAll() ([]User, error)
}

type MySQLUserDAO struct {
    db *sql.DB
}

func (dao *MySQLUserDAO) Insert(user User) error {
    _, err := dao.db.Exec("INSERT INTO users...", user.Name, user.Email)
    return err
}
```

## 6. Query Builder

Конструирует SQL-запросы программно.

```go
type QueryBuilder struct {
    table     string
    selects   []string
    wheres    []string
    whereArgs []interface{}
}

func (qb *QueryBuilder) Select(fields ...string) *QueryBuilder {
    qb.selects = fields
    return qb
}

func (qb *QueryBuilder) Where(condition string, arg interface{}) *QueryBuilder {
    qb.wheres = append(qb.wheres, condition)
    qb.whereArgs = append(qb.whereArgs, arg)
    return qb
}
```

## 7. ORM (Object-Relational Mapping)

Использование библиотек типа GORM, XORM:

```go
import "gorm.io/gorm"

type Product struct {
    gorm.Model
    Code  string
    Price uint
}

func main() {
    db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
    
    // Автомиграция
    db.AutoMigrate(&Product{})
    
    // Создание
    db.Create(&Product{Code: "D42", Price: 100})
    
    // Чтение
    var product Product
    db.First(&product, 1)
}
```

Выбор паттерна зависит от сложности проекта, требований к тестируемости и необходимости абстракции от конкретной СУБД.
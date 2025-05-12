# Рабочий пример Repository Pattern в Go с описанием

Вот полный пример реализации Repository Pattern для работы с пользователями в базе данных:

```go
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"time"

	_ "github.com/lib/pq" // Драйвер PostgreSQL
)

// User - модель данных, представляющая пользователя
type User struct {
	ID        int       `json:"id"`
	Name      string    `json:"name"`
	Email     string    `json:"email"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

// UserRepository - интерфейс репозитория для работы с пользователями
// Определяет контракт для всех операций с пользователями
type UserRepository interface {
	Create(ctx context.Context, user *User) error
	GetByID(ctx context.Context, id int) (*User, error)
	GetByEmail(ctx context.Context, email string) (*User, error)
	Update(ctx context.Context, user *User) error
	Delete(ctx context.Context, id int) error
	List(ctx context.Context, limit, offset int) ([]*User, error)
}

// userRepository - реализация UserRepository для PostgreSQL
type userRepository struct {
	db *sql.DB
}

// NewUserRepository создает новый экземпляр userRepository
func NewUserRepository(db *sql.DB) UserRepository {
	return &userRepository{db: db}
}

// Create добавляет нового пользователя в базу данных
func (r *userRepository) Create(ctx context.Context, user *User) error {
	query := `
		INSERT INTO users (name, email, created_at, updated_at)
		VALUES ($1, $2, $3, $4)
		RETURNING id
	`

	now := time.Now()
	err := r.db.QueryRowContext(
		ctx,
		query,
		user.Name,
		user.Email,
		now,
		now,
	).Scan(&user.ID)

	if err != nil {
		return fmt.Errorf("failed to create user: %w", err)
	}

	user.CreatedAt = now
	user.UpdatedAt = now
	return nil
}

// GetByID получает пользователя по ID
func (r *userRepository) GetByID(ctx context.Context, id int) (*User, error) {
	query := `
		SELECT id, name, email, created_at, updated_at
		FROM users
		WHERE id = $1
	`

	user := &User{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&user.ID,
		&user.Name,
		&user.Email,
		&user.CreatedAt,
		&user.UpdatedAt,
	)

	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("user not found: %w", err)
		}
		return nil, fmt.Errorf("failed to get user: %w", err)
	}

	return user, nil
}

// GetByEmail получает пользователя по email
func (r *userRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
	query := `
		SELECT id, name, email, created_at, updated_at
		FROM users
		WHERE email = $1
	`

	user := &User{}
	err := r.db.QueryRowContext(ctx, query, email).Scan(
		&user.ID,
		&user.Name,
		&user.Email,
		&user.CreatedAt,
		&user.UpdatedAt,
	)

	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("user not found: %w", err)
		}
		return nil, fmt.Errorf("failed to get user by email: %w", err)
	}

	return user, nil
}

// Update обновляет данные пользователя
func (r *userRepository) Update(ctx context.Context, user *User) error {
	query := `
		UPDATE users
		SET name = $1, email = $2, updated_at = $3
		WHERE id = $4
	`

	now := time.Now()
	result, err := r.db.ExecContext(
		ctx,
		query,
		user.Name,
		user.Email,
		now,
		user.ID,
	)

	if err != nil {
		return fmt.Errorf("failed to update user: %w", err)
	}

	rowsAffected, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("failed to get rows affected: %w", err)
	}

	if rowsAffected == 0 {
		return fmt.Errorf("user not found")
	}

	user.UpdatedAt = now
	return nil
}

// Delete удаляет пользователя по ID
func (r *userRepository) Delete(ctx context.Context, id int) error {
	query := `DELETE FROM users WHERE id = $1`

	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return fmt.Errorf("failed to delete user: %w", err)
	}

	rowsAffected, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("failed to get rows affected: %w", err)
	}

	if rowsAffected == 0 {
		return fmt.Errorf("user not found")
	}

	return nil
}

// List возвращает список пользователей с пагинацией
func (r *userRepository) List(ctx context.Context, limit, offset int) ([]*User, error) {
	query := `
		SELECT id, name, email, created_at, updated_at
		FROM users
		ORDER BY id
		LIMIT $1 OFFSET $2
	`

	rows, err := r.db.QueryContext(ctx, query, limit, offset)
	if err != nil {
		return nil, fmt.Errorf("failed to query users: %w", err)
	}
	defer rows.Close()

	var users []*User
	for rows.Next() {
		user := &User{}
		if err := rows.Scan(
			&user.ID,
			&user.Name,
			&user.Email,
			&user.CreatedAt,
			&user.UpdatedAt,
		); err != nil {
			return nil, fmt.Errorf("failed to scan user: %w", err)
		}
		users = append(users, user)
	}

	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("rows error: %w", err)
	}

	return users, nil
}

func main() {
	// Подключение к базе данных PostgreSQL
	db, err := sql.Open("postgres", "host=localhost port=5432 user=postgres password=postgres dbname=test sslmode=disable")
	if err != nil {
		log.Fatalf("failed to connect to database: %v", err)
	}
	defer db.Close()

	// Проверка соединения
	if err := db.Ping(); err != nil {
		log.Fatalf("failed to ping database: %v", err)
	}

	// Создание репозитория
	userRepo := NewUserRepository(db)
	ctx := context.Background()

	// Пример использования репозитория

	// Создание пользователя
	newUser := &User{
		Name:  "John Doe",
		Email: "john@example.com",
	}
	if err := userRepo.Create(ctx, newUser); err != nil {
		log.Printf("Failed to create user: %v", err)
	} else {
		log.Printf("Created user with ID: %d", newUser.ID)
	}

	// Получение пользователя по ID
	user, err := userRepo.GetByID(ctx, newUser.ID)
	if err != nil {
		log.Printf("Failed to get user: %v", err)
	} else {
		log.Printf("Retrieved user: %+v", user)
	}

	// Обновление пользователя
	user.Name = "John Updated"
	if err := userRepo.Update(ctx, user); err != nil {
		log.Printf("Failed to update user: %v", err)
	} else {
		log.Printf("User updated successfully")
	}

	// Получение списка пользователей
	users, err := userRepo.List(ctx, 10, 0)
	if err != nil {
		log.Printf("Failed to list users: %v", err)
	} else {
		log.Printf("Retrieved %d users", len(users))
	}

	// Удаление пользователя
	if err := userRepo.Delete(ctx, user.ID); err != nil {
		log.Printf("Failed to delete user: %v", err)
	} else {
		log.Printf("User deleted successfully")
	}
}
```

## Описание реализации

1. **Модель данных (User)**:
   - Определяет структуру данных пользователя
   - Содержит поля, соответствующие столбцам в БД

2. **Интерфейс UserRepository**:
   - Определяет контракт для работы с пользователями
   - Включает CRUD операции (Create, Read, Update, Delete)
   - Добавлены методы для поиска по email и получения списка

3. **Реализация userRepository**:
   - Инкапсулирует логику работы с БД
   - Использует sql.DB для выполнения запросов
   - Обрабатывает ошибки, включая случай "не найдено"
   - Возвращает понятные ошибки с обертыванием (fmt.Errorf с %w)

4. **Особенности реализации**:
   - Поддержка context для отмены операций
   - Правильное управление соединениями (defer rows.Close())
   - Обработка всех возможных ошибок
   - Использование RETURNING в INSERT для получения ID
   - Проверка RowsAffected при UPDATE/DELETE

5. **Преимущества такого подхода**:
   - Отделение бизнес-логики от деталей работы с БД
   - Возможность легко заменить реализацию (например, на mock для тестов)
   - Единая точка управления операциями с пользователями
   - Упрощение миграции на другую СУБД

## Как использовать

1. Создайте таблицу в PostgreSQL:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

2. Запустите программу, предварительно установив драйвер:
```bash
go get github.com/lib/pq
go run main.go
```

3. Программа продемонстрирует все основные операции:
   - Создание пользователя
   - Получение по ID
   - Обновление данных
   - Получение списка
   - Удаление пользователя

Для тестирования можно легко создать Mock-реализацию UserRepository, что упрощает unit-тестирование кода, который зависит от репозитория.
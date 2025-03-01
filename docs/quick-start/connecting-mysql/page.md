# Connecting MySQL

Just like any other datasource gofr also supports connection to MySQL database based on configuration variables. It automatically manages the connection pool, connection retry etc.

## Setup

You can run the mysql server and create a database locally using the following docker command:

```bash
docker run --name gofr-mysql -e MYSQL_ROOT_PASSWORD=root123 -e MYSQL_DATABASE=test_db -p 3306:3306 -d mysql:8.0.30
```

Access `test_db` database and create table customer with columns `id` and `name`

```bash
docker exec -it gofr-mysql mysql -uroot -proot123 test_db -e "CREATE TABLE customers (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255) NOT NULL);"
```

Now the database with table is ready, we can connect our server to MySQL

## Configuration & Usage

After adding MySQL configs `.env` will be updated to the following.

```bash
# configs/.env
APP_NAME=test-service
HTTP_PORT=9000

REDIS_HOST=localhost
REDIS_PORT=6379

DB_HOST=localhost
DB_USER=root
DB_PASSWORD=root123
DB_NAME=test_db
DB_PORT=3306
```

Now in the following example let's store customer data using **POST** `/customer` and then use **GET** `/customer` to retrieve the same.
We will be storing the customer data with `id` and `name`.

After adding code to add and retrieve data from MySQL datastore `main.go` will be updated to the following.

```go
package main

import "gofr.dev/pkg/gofr"

type Customer struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func main() {
	// initialise gofr object
	app := gofr.New()

	app.GET("/greet", func(ctx *gofr.Context) (interface{}, error) {
		// Get the value using the redis instance
		value, err := ctx.Redis.Get(ctx.Context, "greeting").Result()

		return value, err
	})

	app.POST("/customer/{name}", func(ctx *gofr.Context) (interface{}, error) {
		name := ctx.PathParam("name")

		// Inserting a customer row in database using SQL
		_, err := ctx.DB.ExecContext(ctx, "INSERT INTO customers (name) VALUES (?)", name)

		return nil, err
	})

	app.GET("/customer", func(ctx *gofr.Context) (interface{}, error) {
		var customers []Customer

		// Getting the customer from the database using SQL
		rows, err := ctx.DB.QueryContext(ctx, "SELECT * FROM customers")
		if err != nil {
			return nil, err
		}

		for rows.Next() {
			var customer Customer
			if err := rows.Scan(&customer.ID, &customer.Name); err != nil {
				return nil, err
			}

			customers = append(customers, customer)
		}

		// return the customer
		return customers, nil
	})

	// Starts the server, it will listen on the default port 8000.
	// it can be over-ridden through configs
	app.Run()
}
```

To update the database with the customer data access use through this curl command through terminal

```bash
# here abc and xyz after /customer are the path parameters
curl --location --request POST 'http://localhost:9000/customer/abc'

curl --location --request POST 'http://localhost:9000/customer/xyz'
```

You will see the following output if database is successfully updated

```json
{}
```

Now when we access the [http://localhost:9000/customer](http://localhost:9000/customer) we should see the following output

```json
{
  "data": [
    {
      "id": 1,
      "name": "abc"
    },
    {
      "id": 2,
      "name": "xyz"
    }
  ]
}
```

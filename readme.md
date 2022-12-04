## Project structure

- **bin** contain our compiled binaries, ready for deployment to a production server
- **cmd/api** contain the app-specific code for API. This includes the code for
  running the server, reading and writing HTTP requests and managing auth
- **internal** contain various ancillary packages used by API. It contains the code
  for interacting with db, doing data validation, sending emails and so on. Basically,
  any code which isn't app specific and can potentially be reused lives here. Our
  go code under _cmd/api_ import the packages in the _internal_ directory (but never
  the other way around)
- **migrations** contain the SQL migration files for our db
- **remote** contain the configuration files and setup scripts for our prod server
- **go.mod** declare our project dependencies, versions and module path
- **Makefile** contain _recipes_ for automating common administrative tasks - like
  auditing Go code, building binaries, executing db migrations

## Launching database for local development

- `./initdb.exe -D ../data --username=postgres --auth=trust`
- `./pg_ctl.exe start -D ../data`
- `./psql.exe --username=postgres -p 5434`
- `./psql.exe --username=greenlight --dbname=greenlight -p 5434`

## Working with migrations in 'migrate' tool

- `migrate create -seq -ext=.sql -dir=./migrations create_movies_table` - Create migration
- `migrate -path=./migrations -database=$GREENLIGHT_DB_DSN up` - Run migrations


## Give permission to users

```SQL
-- Set the activated field for alice@example.com to true.
UPDATE users SET activated = true WHERE email = 'alice@example.com';

-- Give all users the 'movies:read' permission
INSERT INTO users_permissions
SELECT id, (SELECT id FROM permissions WHERE code = 'movies:read') FROM users;

-- Give faith@example.com the 'movies:write' permission
INSERT INTO users_permissions
VALUES (
    (SELECT id FROM users WHERE email = 'faith@example.com'),
    (SELECT id FROM permissions WHERE  code = 'movies:write')
);

-- List all activated users and their permissions.
SELECT email, array_agg(permissions.code) as permissions
FROM permissions
INNER JOIN users_permissions ON users_permissions.permission_id = permissions.id
INNER JOIN users ON users_permissions.user_id = users.id
WHERE users.activated = true
GROUP BY email;
```

## Start Project with CORS
- `go run ./cmd/api -cors-trusted-origins="http://localhost:9000 http://localhost:9001"`

## Generate load with 'hey' tool
- https://github.com/rakyll/hey
- `BODY='{"email": "alice@example.com", "password": "pa55word"}'`
- `hey -d "$BODY" -m "POST" http://localhost:4000/v1/tokens/authentication`
- Make sure API has rate limiter turned off with the `-limiter-enabled=false`

## Check source code with 'staticcheck' tool
- `go install honnef.co/go/tools/cmd/staticcheck@latest`

## Disabling GoProxy
- By default: "GOPROXY=https://proxy.golang.org,direct"
- See in `go env`
- Use direct: "export GOPROXY=direct"

## Cross Compilation
- See a list of all the os/architecture combinations that Go supports: `go tool dist list`

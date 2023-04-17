## Подключение к приватному gitlab

1. Добавить в .zshrc

```bash
export GO111MODULE=on
export GOPRIVATE="gitlab.company.com"
```

2. Добавить в ~/.ssh/config

```
Host gitlab.company.com
	Hostname gitlab.company.com
	IdentityFile ~/.ssh/<key_name>
```

3. Выполнить:

```
ssh-keygen -R gitlab.company.com
ssh-keyscan -t rsa gitlab.company.com >> ~/.ssh/known_hosts
git config --global url."git@gitlab.company.com:".insteadOf "https://gitlab.company.com"
```

## Сборка проекта

```bash
go mod download
go mod tidy
go build
```

## Запуск тестов
```bash
go test ./...
```

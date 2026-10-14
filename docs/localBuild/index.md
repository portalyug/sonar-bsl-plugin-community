## Часть 0. Перед началом работ

##### 1. Установить на локальном компьютере Git
##### 2. Установить на локальном компьютере JDK 21

## Часть 1. Синхронизация форков с оригиналами

GitHub не обновляет форки автоматически. Чтобы получить свежий код из оригинальных репозиториев, нужно обновить форки bsl-language-server и sonar-bsl-plugin-community командой "Sync fork" на главной странице репозитория, либо вручную в локальном репозитории добавить `upstream` и смержить.

### 1.1. bsl-language-server

```bash
# Клонируем ваш форк
git clone https://github.com/portalyug/bsl-language-server.git
cd bsl-language-server

# Добавляем оригинальный репозиторий как upstream
git remote add upstream https://github.com/1c-syntax/bsl-language-server.git

# Забираем свежий код из оригинала
git fetch upstream

# Переходим в ветку develop (основная ветка разработки)
git checkout develop

# Мержим изменения из оригинала в ваш форк
git merge upstream/develop

# Если есть конфликты — правим их, затем:
git add .
git commit -m "Merge upstream develop"

# Отправляем обновлённый форк на GitHub
git push origin develop
```

### 1.2. sonar-bsl-plugin-community

```bash
# Клонируем ваш форк плагина
git clone https://github.com/portalyug/sonar-bsl-plugin-community.git
cd sonar-bsl-plugin-community

# Добавляем оригинальный репозиторий как upstream
git remote add upstream https://github.com/1c-syntax/sonar-bsl-plugin-community.git

# Забираем свежий код
git fetch upstream

# Переходим в основную ветку (develop или master — проверьте, какая есть)
git checkout develop
# Если develop нет, попробуйте: git checkout master

# Мержим оригинал
git merge upstream/develop

# При конфликтах — правим, коммитим, пушим
git push origin develop
```
---

## Часть 2. Публикация bsl-language-server в mavenLocal

### 2.1. Создаём тег с версией

Для сборки нужен тег вида `v<версия>`:

```bash
cd bsl-language-server

# Убедимся, что все изменения закоммичены
git status

# Создаём тег. Назовём, например, v1.0.7-fork, где v1.0.7 это текущая актуальная версия bsl-language-server — 
# плагин вытащит "1.0.7-fork" как версию
git tag v1.0.7-fork

# Версия проекта теперь автоматически станет "1.0.7-fork"
```

### 2.2. Публикуем в локальный Maven-репозиторий

```bash
./gradlew publishToMavenLocal
```

Артефакты появятся в:

```
~/.m2/repository/io/github/1c-syntax/bsl-language-server/1.0.7-fork/
```

Там будут: основной JAR, sources JAR, javadoc JAR и boot JAR (с классифайером `exec`).

> **Если сборка падает** — возможно, не хватает JDK 21 (проект требует `JavaVersion.VERSION_21`). Проверьте: `java -version`. При необходимости установите JDK 21 и укажите путь через `JAVA_HOME`.

---

## Часть 3. Сборка sonar-bsl-plugin-community с зависимостью от локального bsl-language-server

### 3.1. Добавляем mavenLocal в репозитории

В файле `build.gradle.kts` плагина найдите блок `repositories`. Если `mavenLocal()` там нет — добавьте его **первым**, чтобы Gradle искал артефакт сначала локально:

```kotlin
repositories {
    mavenLocal()   // ← добавить первой строкой
    mavenCentral()
    // ... остальное, что уже есть
}
```

> `mavenLocal()` уже может присутствовать — bsl-language-server тоже использует его. Проверьте, чтобы строка была в начале списка.

### 3.2. Обновляем зависимости

Найдите строку:

```kotlin
implementation("io.github.1c-syntax:bsl-language-server:1.0.7")
```

Замените версию на вашу:

```kotlin
implementation("io.github.1c-syntax:bsl-language-server:1.0.7-fork")
```

Группа и имя артефакта не меняются — меняется только версия, которая соответствует тегу, который вы создали.


Если при запуске тестов часть падает с ошибкой "Sentry SDK has detected a mix of versions", то необходимо в зависимости добавить строку с актуальной на данный момент версией io.sentry:sentry-bom.

Пример:

```kotlin
implementation(platform("io.sentry:sentry-bom:8.58.0"))
```


### 3.3. Собираем плагин

```bash
cd sonar-bsl-plugin-community
git tag v1.1.1-portalug <- текущая версия плагина
git checkout v1.1.1-portalug
./gradlew build
```

Готовый JAR плагина появится в `build/libs/`.

---

## Шпаргалка: все команды по порядку

```bash
# === 1. Синхронизация bsl-language-server ===
git clone https://github.com/portalyug/bsl-language-server.git
cd bsl-language-server
git remote add upstream https://github.com/1c-syntax/bsl-language-server.git
git fetch upstream
git checkout develop
git merge upstream/develop
git push origin develop

# === 2. Тег и публикация в mavenLocal ===
git tag v1.0.7-fork
./gradlew publishToMavenLocal

# === 3. Синхронизация sonar-bsl-plugin-community ===
cd ..
git clone https://github.com/portalyug/sonar-bsl-plugin-community.git
cd sonar-bsl-plugin-community
git remote add upstream https://github.com/1c-syntax/sonar-bsl-plugin-community.git
git fetch upstream
git checkout develop
git merge upstream/develop
git push origin develop

# === 4. Правим build.gradle.kts ===
# Добавить mavenLocal() в repositories
# Заменить версию: 1.0.7 → 1.0.7-fork

# === 5. Сборка плагина ===
git tag 1.1.1 <- текущая версия плагина
git checkout 1.1.1
./gradlew build
```

---

## Возможные проблемы

| Симптом | Причина | Решение |
|---|---|---|
| `Could not resolve io.github.1c-syntax:bsl-language-server:1.0.7-fork` | Артефакт не опубликован в mavenLocal | Выполните `./gradlew publishToMavenLocal` в репозитории bsl-language-server. Проверьте: `ls ~/.m2/repository/io/github/1c-syntax/bsl-language-server/` |
| Версия определяется как `develop-XXXXX-SNAPSHOT` | HEAD не на теге | `git tag v1.0.7-fork` и убедитесь, что HEAD указывает на коммит с тегом: `git describe --tags` должен вывести `v1.0.7-fork` |
| Сборка требует JDK 21 | Проект использует `JavaVersion.VERSION_21` | Установите JDK 21, задайте `JAVA_HOME` |
| Конфликты при мерже upstream | В форке есть коммиты, которых нет в оригинале | Разрешите конфликты вручную или сделайте `git reset --hard upstream/develop` и заново примените свои правки |

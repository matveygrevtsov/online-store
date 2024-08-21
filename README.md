# online-store

## Задумка
Написать интернет-магазин, состоящий из трёх микрофронтов:
1) `online-store-main` - это хост-приложение (т.е. главное приложение, в которое будут встраиваться остальные приложения)
2) `online-store-cart` - это корзина (ремоут-приложение, которое будет встраиваться в `online-store-main`)
3) `online-store-header` - это хедер  (ремоут-приложение, которое будет встраиваться в `online-store-main`)

При этом `online-store-main` будет оборачивать `online-store-cart` и `online-store-header` в redux-провайдер, что позволит всем трём микрофронтам обмениваться между собой данными. Авторизацию и базу данных будем использовать из firebase. Каждый из микрофронтов будет жить в отдельном репозитории.

## Немного вводной теории
* ModuleFederationPlugin - это плагин веб-пака, который позволяет шерить компоненты, функции, классы и другие данные между независимыми сборками.
* Технология микрофронтов похожа на айфреймы и на npm-модули. Но в айфреймах, в отличии от микрофронтов, сайт загружается целиком, и осуществлять обмен данными с айфреймом довольно проблематично. А в npm-модулях, в отличии от микрофронтов, нужно следить за версией и постоянно обновлять - это не очень удобно, а в микрофронтах все изменения будут подтягиваться автоматически.
* Микрофронты бывают двух типов: хост и ремоут. Хост приложение - это, по-сути, контейнер, который импортирует в себя остальные ремоут-приложения.

## Как подключить ModuleFederationPlugin в webpack.config.js ?
Рассмотрим пример `webpack.config.js`:

```js
const { ModuleFederationPlugin } = require("webpack").container;
const deps = require("./package.json").dependencies;

const moduleFederationConfig = {
  // Имя нашего приложения
  name: "main",
  // Список ремоут-приложений (которые мы импортируем)
  remotes: {
    header: `header@${headerUrl}/remoteEntry.js`, // headerUrl - это домен, на котором "живёт" хедер; header - это name, который мы задали в ремоуте в moduleFederationConfig.
    cart: `cart@${cartUrl}/remoteEntry.js`, // cartUrl - это домен, на котором "живёт" хедер; cart - это name, который мы задали в ремоуте в moduleFederationConfig.
  },
  // Компоненты, которые мы экспортируем
  exposes: {
    "./ProductCounter": "./src/components/ProductCounter/ProductCounter",
  },
  // Библиотеки, которые мы собираемся шерить.
  shared: {
    ...deps,
    react: {
      singleton: true, // Эта опция говорит о том, что мы хотим использовать только один инстанс библиотеки для всех микрофронтов.
      requiredVersion: deps.react, // Определяет минимальную версию общего модуля, которая должна быть загружена.
      eager: true, // Эта опция говорит о том, что библиотека будет сразу загружена при инициализации приложения, а не по требованию.
    },
    "react-dom": {
      singleton: true,
      requiredVersion: deps["react-dom"],
      eager: true,
    },
    "react-router-dom": {
      singleton: true,
      requiredVersion: deps["react-router-dom"],
      eager: true,
    },
    antd: {
      singleton: true,
      requiredVersion: deps.antd,
      eager: true,
    },
  },
};

module.exports = {
  ...
  plugins: [
    ...
    new ModuleFederationPlugin(moduleFederationConfig),
    new ExternalTemplateRemotesPlugin(), // Этот плагин нужен только для хостовых приложений
  ]
};
```

## Шаг№1: подготовка репозиториев
На этом шаге для каждого микрофронта нужно создать отдельный репозиторий. Так же нужно будет создать главный репозиторий-контейнер online-store, который будет включать в себя все репозитории, соответствующие нашим микро-фронтам.

## Шаг№2: Инициализация суб-модулей в github
Клонирум себе главный репозиторий (online-store):
```console
git clone https://github.com/matveygrevtsov/online-store.git
```
И теперь мы должны каждый микрофронт добавить в качестве суб-модуля:
```console
git submodule add https://github.com/matveygrevtsov/online-store-header.git
git submodule add https://github.com/matveygrevtsov/online-store-cart.git
git submodule add https://github.com/matveygrevtsov/online-store-main.git
```
Далее, мы должны запушить эти суб-модули в основной репозиторий online-store-main.

Полезная статья про субмодули в гите: https://git-scm.com/book/ru/v2/Инструменты-Git-Подмодули

Чтобы актуализировать субмодули, нужно использовать команду:
```console
git submodule update --init --force --remote
```

## Шаг№3: Устанавливаем React для каждого микрофронта
Данный шаг рассмотрим на примере `online-shop-main`. Все остальные микрофронты - по аналогии.

**Инициализируем npm:**
```console
npm init
```

**Создаём файл .gitignore со следующим содержимым:**
```console
/dist
/node_modules
```

**Устанавливаем webpack:**
```console
npm install webpack -D
npm install webpack-cli -D
npm install webpack-dev-server -D
npm install html-webpack-plugin -D # Чтобы вебпак мог работать с html-файлами
```

**Устанавливаем typescript**
```console
npm install typescript -D
npm install ts-loader -D # Чтобы вебпак мог билдить ts-файлы
```

**Инициализируем tsconfig.json**
```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "jsx": "react",
    "typeRoots": ["./node_modules/@types", "./src/types/index.d.ts"],
  },
  "include": ["src"]
}
```

**Создаём основной html-файл и заполняем дефолтным содержимым**
Расположение файла: `./public/index.html `

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <base href="/" />
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta
      name="description"
      content="Web site created using create-react-app"
    />
    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```




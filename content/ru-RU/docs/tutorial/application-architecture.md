# Архитектура приложения Electron

Перед погружением в API нужно обсудить два доступных в Electron типа процессов. Они фундаментально различаются и их важно понимать.

## Main и Renderer процессы

В Electron процесс, который запускает скрипт `main` из `package.json` называется **основным процессом**. Скрипт, запущенный в основном процессе может отображать GUI путём создания веб-страниц. В приложениях Electron всегда есть один главный процесс, но не больше.

Так как Electron использует Chromium для отображения веб-страниц, Chromium мульти-процессорная архитектура также используется. Каждая веб-страница электрон выполняется в собственном процессе, который называют **процесс визуализации**.

В нормальных браузерах, веб-страницы обычно выполняются в изолированной среде и им не разрешается доступ к нативным ресурсам. Пользователи Electron'а, однако, имеют право использовать API Node.js на веб-страницах, позволяя взаимодействовать на нижнем уровне операционной системы.

### Различия между основными процессами и процессами визуализации

Основной процесс создает веб-страницы путем создания экземпляров `BrowserWindow`. Каждый экземпляр `BrowserWindow` запускает веб-страницу в процессе визуализации. Когда экземпляр `BrowserWindow` уничтожается, соответствующий процесс визуализации также прекращается.

Основной процесс управляет всеми веб-страницами и процессами их визуализации. Каждый процесс визуализации изолирован и заботится только о веб-странице, работающей в нем.

В веб-страницах вызов интерфейсов API нативного GUI запрещён, поскольку управление ресурсами нативного GUI на веб-страницах довольно опасно и может вызвать утечку ресурсов. Если вы хотите выполнить GUI операции на веб-странице, процесс визуализации веб-страницы должен общаться с основным процессом для запроса выполнения этих операций основного процесса.

> #### Примечание: взаимодействие между процессами
> 
> In Electron, we have several ways to communicate between the main process and renderer processes, such as [`ipcRenderer`](../api/ipc-renderer.md) and [`ipcMain`](../api/ipc-main.md) modules for sending messages, and the [remote](../api/remote.md) module for RPC style communication. Так же доступен FAQ о [том, как обмениваться данными между веб-страницами](../faq.md#how-to-share-data-between-web-pages).

## Использование API Electron

В Electron доступен ряд API-интерфейсов, поддерживающих разработку настольных приложений в обоих процессах - main process и render process. Для доступа к этим процессам API Electron вам необходимо включить в проект следующий модуль:

```javascript
const electron = require('electron')
```

All Electron APIs are assigned a process type. Many of them can only be used from the main process, some of them only from a renderer process, some from both. The documentation for each individual API will state which process it can be used from.

A window in Electron is for instance created using the `BrowserWindow` class. It is only available in the main process.

```javascript
// This will work in the main process, but be `undefined` in a
// renderer process:
const { BrowserWindow } = require('electron')

const win = new BrowserWindow()
```

Since communication between the processes is possible, a renderer process can call upon the main process to perform tasks. Electron comes with a module called `remote` that exposes APIs usually only available on the main process. In order to create a `BrowserWindow` from a renderer process, we'd use the remote as a middle-man:

```javascript
// This will work in a renderer process, but be `undefined` in the
// main process:
const { remote } = require('electron')
const { BrowserWindow } = remote

const win = new BrowserWindow()
```

## Использование API Node.js

Electron exposes full access to Node.js both in the main and the renderer process. This has two important implications:

1) Все API, доступные в Node.js также доступны и в Electron. Вызов следующего кода an Electron app:

```javascript
const fs = require('fs')

const root = fs.readdirSync('/')

// Это будет печатать все файлы на корневом уровне диска,
// либо '/' либо 'C:\'.
console.log(root)
```

Как вы уже могли угадать, это имеет важные последствия для безопасности если вы пытаетесь загрузить удаленный контент. Вы можете найти дополнительную информацию и руководство по загрузке remote контента в нашем [security documentation](./security.md).

2) Вы можете использовать Node.js модули в вашем приложении. Выберите свой любимый npm модуль. npm предлагает в настоящее время крупнейшее в мире хранилище open-source код - возможность использовать хорошо поддержанный и проверенный код, который раньше был зарезервированное для серверных приложений, является одной из ключевых особенностей Electron. Контекст | Контекст запроса.

Например чтобы использовать официальный AWS SDK в вашем приложении, следует сначала установить его как зависимость:

```sh
npm install --save aws-sdk
```

Затем в вашем приложении Electron, подключите и используйте модуль, так же как в Node.js приложение:

```javascript
// A ready-to-use S3 Client
const S3 = require('aws-sdk/clients/s3')
```

Существует одна важная оговорка: Native Node.js модули (то есть модули, которые требуют компиляции собственного кода, прежде чем они могут быть использованы) должны быть скомпилированы для использования с Electron.

The vast majority of Node.js modules are *not* native. Only 400 out of the ~650.000 modules are native. However, if you do need native modules, please consult [this guide on how to recompile them for Electron](./using-native-node-modules.md).
---
title: "Node.js Express Winston 사용하여 로깅하기"
description: Node.js Express application에 Winston을 사용하여 로그를 남겨 봅시다.
slug: node-logs-winston
date: 2023-06-18T13:26:26+09:00
image: /img/node.png
categories:
    - Node
draft: false
---

## Why?
* Node.js에서 [console](https://nodejs.org/api/console.html)에 내장된 전역 객체로 표준 출력 또는 표준 에러 출력으로 로그를 기록하는데 사용합니다.
* `console.log()` 또는 `console.error()`로 출력한 로그는 Node.js application이 실행되는 콘솔에 바로 출력되므로, application이 종료되면 로그는 사라집니다.
* 리다이렉션('>')을 사용하여 파일로 리다이렉트를 할 수는 있지만 로그 레벨이나 형식을 설정하거나, 로그 파일을 자동으로 rotate하는 등의 기능은 제공하지 않습니다.
* 우리는 [Winston](https://github.com/winstonjs/winston)과 같은 로깅 모듈을 사용하여 효과적으로 로그를 파일로 관리할 수 있습니다.
  * 로그 레벨, 형식, transport, rotate

이 글에서는 [Winston](https://github.com/winstonjs/winston)으로 사용해서 Node.js application의 로그를 특정 위치에 저장해보고,  
Docker container로 실행시킬 때 Volume을 사용하여 로컬 호스트에도 로그 파일이 저장되도록 설정해보겠습니다.

## Install
```sh
npm install winston
```

## Use
```js
const winston = require('winston');

const logger = winston.createLogger({
    transports: [
        new winston.transports.Console()
    ]
});

logger.info('Hello world');
// {"level":"info","message":"Hello world"}
```

### format
format은 로그 메시지의 출력 포맷을 결정합니다. 
`winston.format` 모듈은 로그 메시지에 대한 사전 설정된 포맷을 사용할 수도 있고 사용자가 정의한 포맷을 사용할 수도 있습니다.
`winston.format.combine`으로 여러 포맷을 하나의 포맷으로 결합할 수도 있습니다.

```js
const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.label({label: 'service'}),
        winston.format.timestamp({
            format: 'YYYY-MM-DDTHH:mm:ss.SSSZ'
        }),
        winston.format.printf(({level, message, label, timestamp}) => {
            return `${timestamp} [${label}] ${level}: ${message}`
        })
    ),
    transports: [
        new winston.transports.Console()
    ],
});

logger.info('Hello world');
// 2023-01-01T12:00:00.000+09:00 [service] info: Hello world
```
* `label`: 각 메시지에 사용자가 정의한 필드를 추가합니다.
* `timestamp`: 메시지를 받은 timestamp를 추가합니다. timestamp의 format도 정의할 수 있습니다.
* `printf`: 메시지의 최종 출력을 정의합니다.

### transports
로그 메시지가 출력되는 위치를 결정합니다. 로그 파일, 콘솔, 외부 서버 등이 될 수 있고, 여러 개의 transport를 동시에 사용하여 동시에 다른 위치에 메시지를 출력할 수 있습니다.
```js
const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.label({label: 'service'}),
        winston.format.timestamp({
            format: 'YYYY-MM-DDTHH:mm:ss.SSSZ'
        }),
        winston.format.printf(({level, message, label, timestamp}) => {
            return `${timestamp} [${label}] ${level}: ${message}`
        })
    ),
    transports: [
        new winston.transports.File({ filename: 'service.log'}),
    ],
});
    
logger.info('Hello world');
// 2023-01-01T12:00:00.000+09:00 [service] info: Hello world
```
위 예제에서는 'service.log' 파일로 출력하고 있습니다.
이렇게 하면 해당 한 파일에 로그가 계속 쌓이게 되는데 로그 파일의 rotation은 파일이 무제한으로 커지는 것을 방지할 수 있습니다.
우리는 [winston-daily-rotate-file](https://github.com/winstonjs/winston-daily-rotate-file)을 사용해서 파일 rotation을 설정해보겠습니다.

```js
import DailyRotateFile from 'winston-daily-rotate-file';

const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.label({label: 'service'}),
        winston.format.timestamp({
            format: 'YYYY-MM-DDTHH:mm:ss.SSSZ'
        }),
        winston.format.printf(({level, message, label, timestamp}) => {
            return `${timestamp} [${label}] ${level}: ${message}`
        })
    ),
    transports: [
        new DailyRotateFile({
            filename: 'service.log',
            datePattern: 'YYYY-MM-DD_HH',
            maxSize: '20m',
            maxFiles: '7d',
        })
    ],
});

// 파일명 service.log.2023-01-01_12
```
* `filename`: 로깅에 사용할 파일 이름입니다.
* `dataPattern`: rotation에 사용할 moment.js 날짜 포맷을 나타냅니다.
* `maxSize`: rotation할 파일의 최대 크기입니다. 파일의 크기가 maxSize에 도달하면 새로운 파일이 생성됩니다.
* `maxFiles`: 보관할 최대 파일 개수, 일수를 말합니다. 예제의 경우 7일이 경과하면 자동으로 파일이 삭제됩니다.

이 외에도 `zippedArchive` 옵션으로 로그 파일을 압축할 수도, `level`로 로깅 레벨을 설정할 수도 있습니다.
서버 application에서는 access 로그(incoming request에 대한 로그)도 파일로 기록하면 좋겠죠.  
[Morgan](https://github.com/expressjs/morgan)을 winston logger과 함께 사용하면 winston의 기능을 사용해서 로그를 잘 관리할 수 있습니다.

## Morgan
[Morgan](https://github.com/expressjs/morgan)은 HTTP request 로그 미들웨어로 요청, 응답에 대한 정보를 로그로 남길 수 있습니다.
다양한 출력 포맷과 로그 레벨, rotation, 압축을 사용하기 위해 winston 모듈과 함께 사용해보도록 합시다.

```js
const accessLogger = winston.createLogger({
    format: winston.format.combine(
        winston.format.timestamp({
            format: 'YYYY-MM-DDTHH:mm:ss.SSSZ'
        }),
        winston.format.printf(info => {
            let message = info.message;
            if (typeof message === 'object') {
                Object.assign(info, message);
                delete info.message;
            }
            return JSON.stringify(info);
        }),
    ),
    transports: [
        new DailyRotateFile({
            filename: 'access.log',
            datePattern: 'YYYY-MM-DD_HH',
            maxSize: '20m',
            maxFiles: '7d',
        })
    ],
});

morgan.format('json', (tokens, req, res) => {
    return JSON.stringify({
        'ip': tokens['remote-addr'](req, res),
        'method': tokens.method(req, res),
        'url': tokens.url(req, res),
        'status': tokens.status(req, res),
        'contentLength': tokens.res(req, res, 'content-length'),
        'duration': tokens['response-time'](req, res),
        'version': 'HTTP/' + tokens['http-version'](req, res),
        'host': tokens.req(req, res, 'hostname'),
    });
});

const accessStream = {
    write: message => {
        const {ip, method, url, status, contentLength, duration, proto, host} = JSON.parse(message);
        accessLogger.info({ip, method, url, status, contentLength, duration, proto, host});
    }
};

const app = express();
app.use(morgan('json', {stream: accessStream}));

// {"level":"info","timestamp":"2023-01-01T00:00:00.000+09:00","ip":"127.0.0.1","method":"GET","url":"/health","status":"200","duration":"4.900","version":"HTTP/1.1"}
```
write()을 포함한 accessStream을 구현하여 morgan에 설정을 해줍니다.
accessLogger.info()로 추가하게 되면 message의 필드들로 들어가게 되는데 message 밖으로 필드를 꺼내기 위해 `winston.format.printf`를 정의했습니다.
예제에서는 'json'이라는 포맷을 사용자가 정의했지만 이미 정의된 'tiny', 'short', 'common' 등으로 설정하여 보다 간단하게 설정할 수도 있습니다.

## Docker Volumes
express application을 docker container로 실행하면 이 container가 down했을 때 로그 파일도 제거됩니다.
매번 새로운 버전으로 업데이트할 때마다 로그 파일이 제거되면 서비스 운영이 어려움이 있을 것입니다.
이를 방지하기 위해 중요한 로그 데이터는 외부 볼륨에 저장하거나, 별도의 로그 관리 시스템에서 수집할 수 있습니다.  
우리는 docker volumes을 통해 호스트 시스템 디렉토리에 마운트해서 container가 down되더라도 로그 파일이 유지될 수 있도록 설정해보겠습니다.

```yml
version: "3.8"
services:
  backend:
    container_name: backend
    image: backend:${TAG}
    ports:
      - "8080:8080"
    volumes:
      - /local/path/log:/container/path/log
```
호스트(로컬) 경로인 '/local/path/log'를 컨테이너 내부 경로인 '/container/path/log'와 연결하고, 호스트의 로그 파일을 컨테이너의 로그 경로로 마운트합니다.  
이를 통해 로컬 파일 시스템과 컨테이너 간에 데이터를 공유하고, 컨테이너 내부에서 해당 경로에 액세스할 수 있습니다.  
이제는 container가 down되더라도 '/local/path/log'에 로그 파일이 남아 조회해볼 수 있습니다.

## Conclusion
* Node.js에서 winston이라는 사용자 정의 포맷, rotation, 압축 등을 지원하는 로깅 모듈을 통해 다양한 경로로 출력할 수 있습니다.  
* HTTP request, response를 기록하는 morgan도 winston과 함께 사용하면 위 기능들로 로그 파일을 관리할 수 있습니다.  
* docker container로 application을 운영하는 경우 volumes로 로그 파일을 호스트에 마운트하면 container down 시에도 호스트에 로그 파일을 유지할 수 있습니다.

## 참조
* [Github: Winston](https://github.com/winstonjs/winston)
* [Github: winston-daily-rotate-file](https://github.com/winstonjs/winston-daily-rotate-file)
* [Github: Morgan](https://github.com/expressjs/morgan)
* [Docker Docs > Volumes](https://docs.docker.com/storage/volumes/)


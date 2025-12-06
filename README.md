# Методические указания по Асинхронным сервисам

## Задачи:
- Создать асинхронный сервис для выполнения долгой задачи
- Показать межсервисное взаимодействие между новым сервисом и сервисом из прошлых лабораторных

## Создание приложения на NestJS

1.  **Подготовка окружения.** В начале работы необходимо удостовериться в наличии среды выполнения Node.js, проверив версию командой `node -v`; если команда не найдена, следует установить LTS-версию с официального сайта (https://nodejs.org/), после чего глобально инсталлировать утилиту CLI для управления проектами NestJS с помощью команды `npm i -g @nestjs/cli`.

2.  **Инициализация проекта.** Для развертывания базовой структуры приложения выполните в терминале команду `nest new async-service`, при появлении интерактивного запроса выберите пакетный менеджер **npm** (клавиша Enter), что приведет к созданию каталога `async-service` с необходимыми конфигурационными файлами.

3.  **Вход в рабочий каталог.** После успешного создания файлов проекта перейдите в его корневую директорию для выполнения дальнейших команд настройки, введя в терминале команду `cd async-service`.

4.  **Генерация компонентов.** Используя встроенный генератор кода, создайте модуль, контроллер и сервис для сущности "exchange" последовательным выполнением команд `nest g module exchange`, `nest g controller exchange` и `nest g service exchange`, что автоматически сформирует папку `src/exchange` с нужными файлами и подключит их к главному модулю приложения.

5.  **Установка зависимостей.** Для реализации межсервисного взаимодействия и отправки HTTP-запросов к внешнему API установите пакет `axios`, выполнив в терминале команду `npm install axios`.

6.  **Настройка точки входа.** Откройте файл `src/main.ts` и измените функцию `bootstrap`, добавив вызов `app.enableCors()` для разрешения CORS-запросов и изменив при необходимости порт в методе `app.listen(3001)`, чтобы приложение запускалось на порту 3001, но в нашем случае мы не будем менять порт, оставив стандартный 3000

```typescript
// Пример итогового содержимого src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors(); // Разрешаем внешние подключения
  await app.listen(3000); // При необходимости замените порт с 3000 на любой свободный
}
bootstrap();
```

## Синхронный веб-сервер
В этой лабораторной мы хотим показать межсерверное взаимодействие. Мы создадим новый веб-сервер, который будет принимать запросы из БД основного сервера из предыдущих работ. Затем новый веб-сервер "рассчитает" новое значение для какого-то поля этого объекта и отправит PUT-запрос к основному серверу.

В NestJS бизнес-логика (расчеты) размещается в **Service**, а обработка входящих запросов — в **Controller**.

В примере мы будем случайно менять числовое поле `subtotal_rub` валюта после обмена. Функция для "расчётов" `calculateBillValues`:

```typescript
export function calculateBillValues(bills: Bill[], rate: number) {
  return bills.map((b) => ({
    denomination_id: b.denomination_id,
    subtotal_rub: b.denomination * b.count * rate,
  }));
}
```

Для отправки PUT-запроса будем использовать модуль `axios`. В файле `src/exchange/exchange.service.ts` реализуем метод `sendExchangeResult`, который выполняет расчет и отправляет данные ( включая ваш токен ):

```typescript
import { Injectable } from '@nestjs/common';
import axios from 'axios';

export interface Bill { denomination_id: number; denomination: number; count: number; }
export interface RequestData { request_id: string; exchange_rate: number; bills: Bill[]; }

export function calculateBillValues(bills: Bill[], rate: number) {
  return bills.map((b) => ({
    denomination_id: b.denomination_id,
    subtotal_rub: b.denomination * b.count * rate,
  }));
}

@Injectable()
export class ExchangeService {
  private readonly MAIN_SERVICE_URL = 'http://localhost:3001/api/exchange_result';
  private readonly SECRET_TOKEN = 'MY_SECRET_TOKEN'; // Ваш токен

  async sendExchangeResult(data: RequestData) {
    const billvalues = calculateBillValues(data.bills, data.exchange_rate);
    
    await axios.put(this.MAIN_SERVICE_URL, {
      token: this.SECRET_TOKEN,
      request_id: data.request_id,
      breakdown: billvalues,
    });
  }
}
```

Теперь перейдем к контроллеру. В файле `src/stocks/stocks.controller.ts` добавим обработчик POST-метода:
- принимает запрос с полем `pk`
- получает результат "расчётов" из сервиса
- отправляет PUT-запрос к основному серверу (синхронно ожидая результат)

```typescript
import { Controller, Post, Body, Res, HttpStatus } from '@nestjs/common';
import { StocksService } from './stocks.service';
import { Response } from 'express';

@Controller('stocks') // Этот контроллер будет отвечать по пути /stocks
export class StocksController {
  constructor(private readonly stocksService: StocksService) {}

  @Post()
  async setStatus(@Body() body: any, @Res() res: Response) {
    if (body.pk) {
      const id = body.pk;
      const stat = this.stocksService.getRandomStatus();
      
      try {
        // Ждем выполнения запроса к внешнему серверу (синхронное поведение)
        await this.stocksService.sendStatus(id, stat);
        return res.status(HttpStatus.OK).send();
      } catch (e) {
        return res.status(HttpStatus.BAD_REQUEST).send();
      }
    }
    return res.status(HttpStatus.BAD_REQUEST).send();
  }
}
```

Запустим приложение, введя в терминале: `npm run start:dev`.

## Асинхронный веб-сервер

Может так случиться, что дополнительный веб-сервер выполняет расчёты слишком долго. Это можно сымитировать с помощью `setTimeout` (аналог `time.sleep`).

Изменим `src/stocks/stocks.service.ts`. Добавим имитацию долгой работы. В Node.js нет блокирующего `sleep`, поэтому мы используем Promise с таймером. Также объединим расчет и отправку в одну задачу.

```typescript
import { Injectable } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class StocksService {
  private readonly CALLBACK_URL = 'http://127.0.0.1:8000/stocks/';

  // Долгая задача (аналог get_random_status с sleep)
  async calculateAndSend(pk: number) {
    console.log(`Start calculation for ${pk}...`);
    
    // Имитация time.sleep(5)
    await new Promise((resolve) => setTimeout(resolve, 5000));

    const result = {
      id: pk,
      status: Math.random() < 0.5,
    };

    // Вызываем "колбэк" - отправку данных
    await this.statusCallback(result);
  }

  // Аналог status_callback
  private async statusCallback(result: { id: number; status: boolean }) {
    console.log('Calculation finished:', result);
    
    const nurl = `${this.CALLBACK_URL}${result.id}/put/`;
    const answer = { is_growing: result.status };
    
    try {
      await axios.put(nurl, answer, { timeout: 3000 });
      console.log('Callback sent successfully');
    } catch (e) {
      console.error('Error sending callback');
    }
  }
}
```

В таком случае, если бы мы использовали `await` в контроллере, основной сервер завис бы в ожидании на 5 секунд.
Мы хотим, чтобы сервер быстро отвечал на запрос (200 OK), а долгая задача запускалась "в фоновом режиме".

В Python для этого использовался `ThreadPoolExecutor`. В Node.js/NestJS архитектура уже асинхронная (Event Loop). Чтобы запустить задачу в фоне, достаточно просто **вызвать асинхронный метод сервиса, НЕ используя ключевое слово await**.

Обновим `src/stocks/stocks.controller.ts`:

```typescript
import { Controller, Post, Body, Res, HttpStatus } from '@nestjs/common';
import { StocksService } from './stocks.service';
import { Response } from 'express';

@Controller('stocks')
export class StocksController {
  constructor(private readonly stocksService: StocksService) {}

  @Post()
  setStatus(@Body() body: any, @Res() res: Response) {
    if (body.pk) {
      const id = body.pk;

      // ЗАПУСК В ФОНОВОМ РЕЖИМЕ:
      // Мы вызываем метод, но НЕ пишем перед ним await.
      // Код пойдет дальше мгновенно, а метод продолжит выполняться в фоне.
      this.stocksService.calculateAndSend(id);

      // Мгновенно возвращаем 200 OK
      return res.status(HttpStatus.OK).send();
    }
    return res.status(HttpStatus.BAD_REQUEST).send();
  }
}
```

Теперь наш обработчик POST-запроса запустит задачу `calculateAndSend` и тут же продолжит выполнение (отправит ответ 200 OK основному серверу). А спустя 5 секунд сработает логика внутри сервиса и отправит PUT-запрос.

### Полный код файлов для проверки

**src/stocks/stocks.service.ts**:
```typescript
import { Injectable } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class StocksService {
  private readonly CALLBACK_URL = 'http://127.0.0.1:8000/stocks/';

  async calculateAndSend(pk: number) {
    // Имитация задержки 5 секунд
    await new Promise((resolve) => setTimeout(resolve, 5000));

    const result = {
      id: pk,
      status: Math.random() < 0.5,
    };

    // Отправка результата
    const nurl = `${this.CALLBACK_URL}${result.id}/put/`;
    const answer = { is_growing: result.status };
    
    try {
      await axios.put(nurl, answer, { timeout: 3000 });
      console.log(`Success update for id ${pk}: ${result.status}`);
    } catch (e) {
      console.error(`Error updating id ${pk}`);
    }
  }
}
```

**src/stocks/stocks.controller.ts**:
```typescript
import { Controller, Post, Body, Res, HttpStatus } from '@nestjs/common';
import { StocksService } from './stocks.service';
import { Response } from 'express';

@Controller('stocks')
export class StocksController {
  constructor(private readonly stocksService: StocksService) {}

  @Post()
  setStatus(@Body() body: any, @Res() res: Response) {
    if (body && body.pk) {
      // Fire-and-forget (запуск без ожидания)
      this.stocksService.calculateAndSend(body.pk);
      
      return res.status(HttpStatus.OK).send();
    }
    return res.status(HttpStatus.BAD_REQUEST).send();
  }
}
```

## Тестирование

Проверим работу асинхронного сервера. 

1. Убедитесь, что ваш основной сервер Django (на порту 8000) запущен.
2. Запустите NestJS сервер: `npm run start:dev` (он запустится на порту 3001).
3. Отправляем запрос на NestJS.
   
   Пример через curl:
   ```bash
   curl -X POST http://localhost:3001/stocks \
     -H "Content-Type: application/json" \
     -d '{"pk": 1}'
   ```
   
   Вам **мгновенно** придет ответ со статусом 200 OK.

4. Спустя примерно 5 секунд в терминале NestJS появится сообщение об успешной отправке, а данные на основном сервере (Django) обновятся.

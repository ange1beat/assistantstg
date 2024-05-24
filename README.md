# Документация коду Telegram-бота на Node.js

## Описание
Этот проект представляет собой Telegram-бота, созданного с использованием Node.js, Express.js, Mongoose и OpenAI API. Бот предоставляет пользователям возможность взаимодействовать с персонажами (вайфу) через чат, отправлять запросы и получать ответы от AI.

## Установка и настройка

### Установка зависимостей
Для начала, установите необходимые зависимости с помощью npm:
```bash
npm install express body-parser mongoose node-telegram-bot-api dotenv openai
```

### Настройка переменных окружения
Создайте файл `.env` в корне проекта и добавьте следующие переменные окружения:
```
MONGODB_URI=ваш_URI_для_MongoDB
TELEGRAM_TOKEN=ваш_Telegram_токен
OPENAI_API_KEY=ваш_API_ключ_OpenAI
PAYMENT_PROVIDER_TOKEN=ваш_токен_платежного_провайдера
```

## Структура проекта

### Основные зависимости
```javascript
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const TelegramBot = require('node-telegram-bot-api');
const dotenv = require('dotenv');
const { OpenAI } = require('openai');
dotenv.config();
```

### Инициализация приложения
```javascript
const app = express();
app.use(bodyParser.json());
```

### Настройка OpenAI
```javascript
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

### Подключение к MongoDB
```javascript
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => {
  console.log('Connected to MongoDB');
}).catch((err) => {
  console.error('Error connecting to MongoDB', err);
});
```

### Определение схемы пользователя и модели
```javascript
const userSchema = new mongoose.Schema({
  chatId: { type: String, required: true },
  remainingRequests: { type: Number, default: 0 },
  lastPaymentDate: { type: Date, default: null },
  threads: {
    Yuki: { type: String, default: null },
    Aska: { type: String, default: null },
  },
  character: { type: String, default: 'Yuki' },
  dailyRequestCount: { type: Number, default: 0 },
  lastRequestDate: { type: Date, default: null },
  requestPending: { type: Boolean, default: false }
});

const User = mongoose.model('User', userSchema);
```

### Инициализация Telegram-бота
```javascript
const bot = new TelegramBot(process.env.TELEGRAM_TOKEN, { polling: true });
```

### Опции покупки запросов
```javascript
const purchaseOptions = {
  '30': { price: 9900, description: '50 запросов' },
  '100': { price: 19900, description: '130 запросов' },
  '200': { price: 29900, description: '300 запросов' }
};
```

### Генерация клавиатуры выбора персонажа
```javascript
const generateCharacterSelectionKeyboard = (user) => {
  const keyboard = [
    [{ text: 'Юки', callback_data: 'character_Yuki' }],
    [{ text: 'Аска', callback_data: 'character_Aska' }]
  ];

  keyboard.forEach(row => {
    row.forEach(button => {
      if (button.callback_data === `character_${user.character}`) {
        button.text = `✅ ${button.text}`;
      }
    });
  });

  return keyboard;
};
```

### Обработка команд и сообщений

#### Команда /custom
Отправляет сообщение с предложением добавления собственного персонажа.
```javascript
bot.onText(/\/custom/, async (msg) => {
  const chatId = msg.chat.id.toString();

  bot.sendMessage(chatId, 'Хотите добавить свою вайфу? Пишите менджеру, стоимсть 99 рублей: [Менеджер](https://t.me/fron1runner)', {
    parse_mode: 'Markdown'
  });
});
```

#### Команда /start
Инициализация пользователя в базе данных и отправка приветственного сообщения.
```javascript
bot.onText(/\/start/, async (msg) => {
  const chatId = msg.chat.id.toString();

  try {
    let user = await User.findOne({ chatId: chatId });

    if (!user) {
      user = new User({ chatId: chatId });
      await user.save();
    }

    await bot.sendPhoto(chatId, 'https://i.postimg.cc/rwg95dxZ/Waifu-Gram-AI.png', {
      caption: 'Добро пожаловать! ✨\n\nИспользуйте кнопку ниже для выбора вайфу.',
      parse_mode: 'Markdown',
      reply_markup: {
        inline_keyboard: [
          [{ text: 'Выбрать вайфу', callback_data: 'select_character' }]
        ]
      }
    });
  } catch (error) {
    console.error('Ошибка обработки команды /start:', error);
    await bot.sendMessage(chatId, 'Произошла ошибка при запуске бота. Пожалуйста, попробуйте позже.');
  }
});
```

#### Команда /characters
Позволяет пользователю выбрать персонажа для общения.
```javascript
bot.onText(/\/characters/, async (msg) => {
  const chatId = msg.chat.id.toString();

  try {
    let user = await User.findOne({ chatId: chatId });

    bot.sendMessage(chatId, 'Выберите персонажа, с которым хотите общаться:', {
      reply_markup: {
        inline_keyboard: generateCharacterSelectionKeyboard(user)
      }
    });
  } catch (error) {
    console.error('Ошибка обработки команды /characters:', error);
    await bot.sendMessage(chatId, 'Произошла ошибка при обработке команды выбора персонажа. Пожалуйста, попробуйте позже.');
  }
});
```

#### Команда /channel
Отправляет ссылку на новостной канал бота.
```javascript
bot.onText(/\/channel/, (msg) => {
  const chatId = msg.chat.id.toString();
  bot.sendMessage(chatId, 'Присоединяйтесь к нашему новостному каналу: [Новости бота](https://t.me/animegptai)', {
    parse_mode: 'Markdown'
  });
});
```

### Обработка callback_query
Обрабатывает выбор персонажа и покупку запросов.
```javascript
bot.on('callback_query', async (query) => {
  const chatId = query.message.chat.id;
  const action = query.data;

  if (action === 'select_character') {
    try {
      let user = await User.findOne({ chatId: chatId });

      bot.sendMessage(chatId, 'Выберите персонажа, с которым хотите общаться:', {
        reply_markup: {
          inline_keyboard: generateCharacterSelectionKeyboard(user)
        }
      });
    } catch (error) {
      console.error('Ошибка обработки команды выбора персонажа:', error);
      await bot.sendMessage(chatId, 'Произошла ошибка при обработке команды выбора персонажа. Пожалуйста, попробуйте позже.');
    }
    return;
  }

  if (action.startsWith('character_')) {
    const selectedCharacter = action.split('_')[1];
    try {
      let user = await User.findOne({ chatId: chatId });

      if (user) {
        user.character = selectedCharacter;

        if (!user.threads[selectedCharacter]) {
          const myThread = await openai.beta.threads.create();
          user.threads[selectedCharacter] = myThread.id;
        }

        await user.save();

        let imageUrl;
        if (selectedCharacter === 'Yuki') {
          imageUrl = 'https://pibig.info/uploads/posts/2022-11/1669247931_3-pibig-info-p-oregairu-instagram-3.jpg';
        } else if (selectedCharacter === 'Aska') {
          imageUrl = 'https://celes.club/uploads/posts/2022-06/1654285396_1-celes-club-p-aska-yevangelion-oboi-krasivie-1.png';
        }

        await bot.sendPhoto(chatId, imageUrl, {
          caption: `Вы выбрали ${selectedCharacter}. Теперь вы можете задать мне любой вопрос!`,
        });
      } else {
        console.error(`Пользователь с chatId ${chatId} не найден`);
        await bot.sendMessage(chatId, 'Произошла ошибка при выборе персонажа. Пожалуйста, попробуйте позже.');
      }
    } catch (error) {
      bot.sendMessage(chatId, 'Произошла ошибка при выборе персонажа. Пожалуйста, попробуйте позже.');
      console.error('Ошибка обработки выбора персонажа:', error);
    }
    return;
  }

  if (action.startsWith('buy_')) {
    const purchaseOption = action.split('_')[1];
    const { price, description } = purchaseOptions[purchaseOption];
    try {
      await bot.sendInvoice(chatId, 'Покупка запросов', description, `purchase_${purchaseOption}`, process.env.PAYMENT_PROVIDER_TOKEN, 'RUB', [{ label: description, amount: price }], {
        photo_url: 'https://i.postimg.cc/rFVjsCtR/2024-05-21-02-56-48.jpg',
       

 need_name: false,
        need_phone_number: false,
        need_email: false,
        is_flexible: false
      });
    } catch (error) {
      console.error('Ошибка при отправке счета:', error);
      await bot.sendMessage(chatId, 'Произошла ошибка при создании платежа. Пожалуйста, попробуйте позже.');
    }
  }
});
```

### Обработка успешного платежа
```javascript
bot.on('successful_payment', async (msg) => {
  const chatId = msg.chat.id.toString();
  const payload = msg.successful_payment.invoice_payload;
  const purchaseOption = payload.split('_')[1]; // Определяем вариант покупки

  let additionalRequests = 0;
  if (purchaseOption === '30') additionalRequests = 50;
  if (purchaseOption === '100') additionalRequests = 130;
  if (purchaseOption === '200') additionalRequests = 300;

  try {
    let user = await User.findOne({ chatId: chatId });

    if (user) {
      user.remainingRequests += additionalRequests;
      user.lastPaymentDate = new Date();
      await user.save();

      bot.sendMessage(chatId, `✅ Оплата прошла успешно! Теперь у вас есть ${user.remainingRequests} запросов.`);
    } else {
      console.error(`Пользователь с chatId ${chatId} не найден`);
    }
  } catch (error) {
    console.error('Ошибка обработки успешного платежа:', error);
  }
});
```

### Обработка сообщений от пользователей
```javascript
bot.on('message', async (msg) => {
  const chatId = msg.chat.id.toString();
  const text = msg.text;

  // Проверка, является ли сообщение командой
  if (text && text.startsWith('/')) {
    return; // Если сообщение является командой, ничего не делаем
  }

  // Обработка сообщения как запроса к ассистенту
  const today = new Date().setHours(0, 0, 0, 0);

  try {
    let user = await User.findOne({ chatId: chatId });

    if (!user) {
      await bot.sendMessage(chatId, 'Пожалуйста, начните с команды /start.');
      return;
    }

    if (user.requestPending) {
      await bot.sendMessage(chatId, 'Ваш предыдущий запрос еще обрабатывается. Пожалуйста, подождите.');
      return;
    }

    if (user.lastRequestDate && new Date(user.lastRequestDate).setHours(0, 0, 0, 0) !== today) {
      user.dailyRequestCount = 0;
    }

    if (user.dailyRequestCount >= 5 && user.remainingRequests <= 0) {
      await bot.sendMessage(chatId, 'Вы достигли лимита в 5 бесплатных запросов на сегодня! Но вы можете купить еще!', {
        reply_markup: {
          inline_keyboard: [
            [{ text: '50 запросов - 99 рублей', callback_data: 'buy_30' }],
            [{ text: '130 запросов - 199 рублей', callback_data: 'buy_100' }],
            [{ text: '300 запросов - 299 рублей', callback_data: 'buy_200' }]
          ]
        }
      });
      return;
    }

    let assistantId, threadId;
    if (user.character === 'Aska') {
      assistantId = 'asst_SzERNegFnerzKByVBr4AKaeV'; // Замените на реальный ID ассистента для Aska
      threadId = user.threads.Aska;
    } else {
      assistantId = 'asst_516uV7FBGWuUgc9kdzFt9tte'; // Замените на реальный ID ассистента для Yuki
      threadId = user.threads.Yuki;
    }

    if (!threadId) {
      const myThread = await openai.beta.threads.create();
      threadId = myThread.id;

      if (user.character === 'Aska') {
        user.threads.Aska = threadId;
      } else {
        user.threads.Yuki = threadId;
      }
      await user.save();
    }

    user.requestPending = true; // Устанавливаем флаг запроса в true
    await user.save();

    await bot.sendChatAction(chatId, 'typing');
    await openai.beta.threads.messages.create(threadId, {
      role: "user",
      content: text,
    });
    const myRun = await openai.beta.threads.runs.create(threadId, {
      assistant_id: assistantId,
    });

    const retrieveRun = async () => {
      let keepRetrievingRun;

      while (myRun.status === "queued" || myRun.status === "in_progress") {
        keepRetrievingRun = await openai.beta.threads.runs.retrieve(threadId, myRun.id);

        if (keepRetrievingRun.status === "completed") {
          const allMessages = await openai.beta.threads.messages.list(threadId);

          let AssistantVcRuResponse = allMessages.data[0].content[0].text.value;

          AssistantVcRuResponse = AssistantVcRuResponse;

          await bot.sendMessage(chatId, `💬 <b>${user.character}:</b> ${AssistantVcRuResponse}`, { parse_mode: "HTML" });

          user.dailyRequestCount += 1;
          if (user.remainingRequests > 0) {
            user.remainingRequests -= 1; // Уменьшаем количество оставшихся оплаченных запросов
          }
          user.lastRequestDate = new Date();
          user.requestPending = false; // Сбрасываем флаг запроса
          await user.save();

          break;
        }
      }
    };
    retrieveRun();
  } catch (err) {
    console.error('Запрос не удался:', err);
    await bot.sendMessage(chatId, 'Произошла ошибка при обработке вашего запроса. Пожалуйста, попробуйте позже.');
    
    let user = await User.findOne({ chatId: chatId });
    if (user) {
      user.requestPending = false; // Сбрасываем флаг запроса в случае ошибки
      await user.save();
    }
  }
});
```

### Админ панель и рассылка

#### Команда /admin
Открывает доступ к админ-панели.
```javascript
bot.onText(/\/admin/, (msg) => {
  const chatId = msg.chat.id.toString();

  if (chatId !== '5940288643') {
    bot.sendMessage(chatId, 'У вас нет доступа к этой команде.');
    return;
  }

  bot.sendMessage(chatId, 'Добро пожаловать в админ-панель.', {
    reply_markup: {
      inline_keyboard: [
        [{ text: 'Рассылка', callback_data: 'broadcast' }]
      ]
    }
  });
});
```

#### Рассылка сообщений пользователям
```javascript
bot.on('callback_query', async (query) => {
  const chatId = query.message.chat.id;
  const action = query.data;

  if (action === 'broadcast') {

    bot.sendMessage(chatId, 'Пожалуйста, введите сообщение для рассылки.');
    bot.once('message', async (msg) => {
      if (msg.chat.id.toString() === '5940288643') {
        broadcastMessage = msg.text;
        bot.sendMessage(chatId, 'Вы уверены, что хотите разослать это сообщение всем пользователям?', {
          reply_markup: {
            inline_keyboard: [
              [{ text: 'Подтвердить', callback_data: 'confirm_broadcast' }]
            ]
          }
        });
      }
    });
  }

  if (action === 'confirm_broadcast') {

    const users = await User.find();
    for (const user of users) {
      bot.sendMessage(user.chatId, broadcastMessage);
    }

    bot.sendMessage(chatId, 'Сообщение разослано всем пользователям.');
  }
});
```

### Эндпоинт для проверки состояния бота
```javascript
app.get('/', (req, res) => {
  res.send('Bot is running');
});
```

### Запуск сервера
```javascript
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
```

## Заключение
Этот код представляет собой основу для создания Telegram-бота, который использует MongoDB для хранения данных о пользователях, OpenAI API для обработки запросов и взаимодействия с пользователями, а также предоставляет админ панель для управления рассылками. Бот поддерживает функционал покупки дополнительных запросов через встроенные платежи.

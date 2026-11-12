Представленный код на Node.js является реализацией прокси-сервера, предназначенного для трансляции запросов, выполненных в формате OpenAI API, в формат, понятный API российской нейросетевой модели GigaChat от Сбера. Данный инструмент решает актуальную задачу совместимости, позволяя разработчикам, уже использующим в своих приложениях стек технологий OpenAI, с минимальными изменениями переключаться на отечественный аналог. Актуальность такого решения обусловлена стремительным развитием глобального и локального рынков искусственного интеллекта, а также необходимостью адаптации программных продуктов к различным API. Целью данного отчета является детальный анализ архитектуры, функциональности и назначения указанного прокси-сервера, а также его позиционирование в контексте последних достижений в области ИИ-моделей, таких как DeepSeek, GigaChat и ChatGPT.

Вот код в файле gigachat-proxy.js , затем вписать свой ключ запустить node gigachat-proxy.js в командной строке. Добавить в AnythingLLM http://localhost:8500/v1 и выбрать модель.

```javascript

const http = require('http');
const https = require('https');
const { randomUUID } = require('crypto'); // Модуль для генерации уникальных RqUID

const AUTH_KEY = '________ВАШ КЛЮЧ АПИ __________';
const PORT = 8500;

// Глобальные переменные для кэширования токена Сбера
let cachedToken = null;
let tokenExpiresAt = 0;

// Карта соответствия: [Имя в интерфейсе AnythingLLM] -> [Реальный ID модели в API Сбера]
const MODEL_MAPPING = {
    "GigaChat-2": "GigaChat-2",
    "GigaChat-2-Max": "GigaChat-2-Max",
    "GigaChat-2-Pro": "GigaChat-2-Pro",
    "GigaChat-3-Lightning": "GigaChat-3-Lightning",          
    "GigaChat-3-Pro": "GigaChat-3-Pro",
    "GigaChat-3-Ultra": "GigaChat-3-Ultra"
};

// Функция получения токена с поддержкой кэширования и динамическим RqUID
function getAccessToken() {
    return new Promise((resolve, reject) => {
        const currentTime = Date.now();
        
        // Если токен есть в кэше и он еще годен (с запасом в 1 минуту), возвращаем его
        if (cachedToken && currentTime < (tokenExpiresAt - 60000)) {
            return resolve(cachedToken);
        }

        console.log("[OAuth] Кэш пуст или истек. Запрашиваем новый токен у Сбербанка...");

        const req = https.request({
            hostname: 'ngw.devices.sberbank.ru',
            port: 9443,
            path: '/api/v2/oauth',
            method: 'POST',
            rejectUnauthorized: false,
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
                'Accept': 'application/json',
                'RqUID': randomUUID(), // Динамический UUID для каждого запроса авторизации
                'Authorization': `Basic ${AUTH_KEY}`
            }
        }, (res) => {
            let data = '';
            res.on('data', chunk => data += chunk);
            res.on('end', () => {
                try {
                    const json = JSON.parse(data);
                    if (json.access_token) {
                        // Сохраняем токен и время его деградации (обычно выдается на 30 минут)
                        cachedToken = json.access_token;
                        tokenExpiresAt = json.expires_at || (Date.now() + 25 * 60 * 1000); 
                        
                        console.log("[OAuth] Токен успешно обновлен и сохранен в кэш.");
                        resolve(cachedToken);
                    } else {
                        reject(data);
                    }
                } catch(e) { 
                    reject(data); 
                }
            });
        });
        
        req.on('error', reject);
        req.write('scope=GIGACHAT_API_PERS');
        req.end();
    });
}

const server = http.createServer((req, res) => {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', '*');
    res.setHeader('Access-Control-Allow-Headers', '*');
    if (req.method === 'OPTIONS') { res.writeHead(200); res.end(); return; }

    if (req.url === '/v1/models' || req.url === '/models') {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        return res.end(JSON.stringify({
            object: "list",
            data: Object.keys(MODEL_MAPPING).map(id => ({ id: id, object: "model", owned_by: "sberbank" }))
        }));
    }

    if (req.url === '/v1/chat/completions' || req.url === '/chat/completions') {
        let body = '';
        req.on('data', chunk => body += chunk);
        req.on('end', () => {
            getAccessToken().then(token => {
                const parsedBody = JSON.parse(body);
                const isStreamRequested = parsedBody.stream === true;
                
                const uiModelName = parsedBody.model || "GigaChat";
                const officialSberModel = MODEL_MAPPING[uiModelName] || "GigaChat";
                
                parsedBody.model = officialSberModel; 

                if (parsedBody.messages) {
                    parsedBody.messages.forEach(msg => {
                        if (msg.role === 'developer') msg.role = 'system';
                    });
                }

                const sberReq = https.request({
                    hostname: 'gigachat.devices.sberbank.ru',
                    path: '/api/v1/chat/completions',
                    method: 'POST',
                    rejectUnauthorized: false,
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${token}`,
                        ...(isStreamRequested ? { 'Accept': 'text/event-stream' } : { 'Accept': 'application/json' })
                    }
                }, (sberRes) => {
                    if (sberRes.statusCode !== 200) {
                        res.writeHead(sberRes.statusCode, { 'Content-Type': 'application/json' });
                        return res.end(JSON.stringify({ error: { message: `Sberbank error (${sberRes.statusCode}) для модели ${officialSberModel}` } }));
                    }

                    if (isStreamRequested) {
                        res.writeHead(200, {
                            'Content-Type': 'text/event-stream',
                            'Cache-Control': 'no-cache',
                            'Connection': 'keep-alive'
                        });

                        sberRes.on('data', (chunk) => {
                            let chunkStr = chunk.toString();
                            let modifiedChunk = chunkStr.replace(/"model":"[^"]+"/, `"model":"${uiModelName}"`);
                            res.write(modifiedChunk);
                        });

                        sberRes.on('end', () => {
                            res.end();
                            console.log(`[OK] Стриминг модели ${uiModelName} (как ${officialSberModel}) успешно завершен!`);
                        });
                    } else {
                        let sberData = '';
                        sberRes.on('data', c => sberData += c);
                        sberRes.on('end', () => {
                            try {
                                const sberJson = JSON.parse(sberData);
                                const textContent = sberJson.choices && sberJson.choices[0] && sberJson.choices[0].message ? sberJson.choices[0].message.content : "";
                                
                                const openaiResp = {
                                    id: sberJson.id || "chatcmpl-default",
                                    object: "chat.completion",
                                    created: sberJson.created || Math.floor(Date.now() / 1000),
                                    model: uiModelName,
                                    choices: [{
                                        index: 0,
                                        message: { role: "assistant", content: textContent },
                                        finish_reason: "stop"
                                    }],
                                    usage: sberJson.usage || { prompt_tokens: 0, completion_tokens: 0, total_tokens: 0 }
                                };
                                res.writeHead(200, { 'Content-Type': 'application/json' });
                                res.end(JSON.stringify(openaiResp));
                                console.log(`[OK] Успешный JSON ответ от ${uiModelName} отправлен!`);
                            } catch(e) {
                                res.writeHead(500, { 'Content-Type': 'application/json' });
                                res.end(JSON.stringify({ error: { message: "JSON parsing error", detail: String(e) } }));
                            }
                        });
                    }
                });
                sberReq.write(JSON.stringify(parsedBody));
                sberReq.end();
            }).catch(err => {
                res.writeHead(500, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: { message: "OAuth failed", detail: String(err) } }));
            });
        });
    } else {
        res.writeHead(404); res.end();
    }
});

server.listen(PORT, () => console.log(`Node Proxy (Smart Model Mapping Edition) running on port ${PORT}`));

```









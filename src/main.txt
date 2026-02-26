#include <Arduino.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <TFT_eSPI.h>

// --- НАСТРОЙКИ ---
const char* ssid = "They_killed_Kenny";
const char* password = "1975maxx";

TFT_eSPI tft = TFT_eSPI();

// Параметры графика
const int maxPoints = 50;        // Количество точек на графике
float priceHistory[maxPoints];   // Массив истории цен
int chartX = 10;                 // Отступ слева
int chartY = 65;                 // Подняли график повыше (было 80)
int chartW = 140;                // Растянули почти на всю ширину (было 108)
int chartH = 50;

unsigned long lastUpdate = 0;
const long interval = 15000;     // Обновление каждые 15 секунд

// Функция отрисовки графика
void drawChart(uint16_t color) {
    // 1. Ищем min/max для масштабирования
    float minP = priceHistory[0];
    float maxP = priceHistory[0];
    for (int i = 1; i < maxPoints; i++) {
        if (priceHistory[i] < minP) minP = priceHistory[i];
        if (priceHistory[i] > maxP) maxP = priceHistory[i];
    }

    // Очищаем только область графика
    tft.fillRect(chartX, chartY, chartW, chartH, TFT_BLACK);
    // Рамка (опционально)
    tft.drawRect(chartX-1, chartY-1, chartW+2, chartH+2, TFT_DARKGREY);

    if (maxP == minP) return;

    // 2. Рисуем линии
    for (int i = 0; i < maxPoints - 1; i++) {
        // Если данных еще нет (0), не рисуем
        if (priceHistory[i] == 0 || priceHistory[i+1] == 0) continue;

        int x1 = chartX + (i * chartW) / (maxPoints - 1);
        int x2 = chartX + ((i + 1) * chartW) / (maxPoints - 1);
        int y1 = chartY + chartH - (int)((priceHistory[i] - minP) * chartH / (maxP - minP));
        int y2 = chartY + chartH - (int)((priceHistory[i+1] - minP) * chartH / (maxP - minP));

        tft.drawLine(x1, y1, x2, y2, color);
    }
}

void getPrice() {
    if (WiFi.status() == WL_CONNECTED) {
        HTTPClient http;
        // Используем API Binance
        http.begin("https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT");
        
        int httpCode = http.GET();
        if (httpCode == 200) {
            String payload = http.getString();
            StaticJsonDocument<200> doc;
            deserializeJson(doc, payload);
            
            float currentPrice = doc["price"].as<float>();

            // Сдвигаем историю
            for (int i = 0; i < maxPoints - 1; i++) {
                priceHistory[i] = priceHistory[i + 1];
            }
            priceHistory[maxPoints - 1] = currentPrice;

            // Определяем цвет (сравнение с предыдущей точкой)
            uint16_t color = (priceHistory[maxPoints-1] >= priceHistory[maxPoints-2]) ? TFT_GREEN : TFT_RED;

            // --- ОТРИСОВКА ТЕКСТА ---
            tft.setTextDatum(TL_DATUM);
            tft.setTextColor(TFT_WHITE, TFT_BLACK); // Фон черный (убирает наложение)
            tft.drawString("BTC / USDT", 10, 10, 2);

            // Отрисовка цены крупно
            tft.setTextColor(color, TFT_BLACK);
            char priceStr[15];
            snprintf(priceStr, sizeof(priceStr), "%.2f  ", currentPrice); // Пробелы в конце затирают старые знаки
            tft.drawString(priceStr, 10, 35, 4);

            // Отрисовка графика
            drawChart(color);
        }
        http.end();
    }
}

void setup() {
    Serial.begin(115200);
    tft.init();
    tft.setRotation(1); // Альбомная ориентация
    tft.fillScreen(TFT_BLACK);
    
    // Инициализация массива нулями
    for(int i=0; i<maxPoints; i++) priceHistory[i] = 0;

    tft.drawString("Connecting to WiFi...", 10, 10, 2);
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    tft.fillScreen(TFT_BLACK);
    getPrice();
}

void loop() {
    unsigned long now = millis();
    if (now - lastUpdate > interval) {
        lastUpdate = now;
        getPrice();
    }
}
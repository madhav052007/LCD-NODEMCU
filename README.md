# LCD-NODEMCU
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <LiquidCrystal.h>

// CHANGE YOUR WIFI CREDENTIALS HERE
const char* ssid = "Enthusiasts";
const char* password = "madhav1234";

LiquidCrystal lcd(0, 16, 14, 12, 13, 5);  // RS=D3, E=D0, D4=D5, D5=D6, D6=D7, D7=D1

ESP8266WebServer server(80);

// Full messages (can be longer than 16 chars)
String fullLine1 = "ESP8266 Ready   ";
String fullLine2 = "Type message!   ";

// Scrolling state
int pos1 = 0, pos2 = 0;
unsigned long lastScroll = 0;
const unsigned long scrollDelay = 700;   // SLOWER SCROLL (700 ms) - change this number only

void setup() {
  Serial.begin(115200);

  lcd.begin(16, 2);
  lcd.print("Connecting WiFi");

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  lcd.clear();
  lcd.print("WiFi Connected");
  lcd.setCursor(0, 1);
  lcd.print(WiFi.localIP());

  Serial.println();
  Serial.print("Open in browser: http://");
  Serial.println(WiFi.localIP());

  delay(2000);
  updateDisplay();

  server.on("/", handleRoot);
  server.on("/set", handleSet);
  server.begin();
  Serial.println("Web server started");
}

void loop() {
  server.handleClient();

  // Scroll engine
  if (millis() - lastScroll >= scrollDelay) {
    lastScroll = millis();
    scrollIfNeeded();
  }
}

void updateDisplay() {
  lcd.clear();

  // Line 1
  if (fullLine1.length() > 16) {
    lcd.print(fullLine1.substring(pos1, pos1 + 16));
  } else {
    lcd.print(fullLine1);
  }

  // Line 2
  lcd.setCursor(0, 1);
  if (fullLine2.length() > 16) {
    lcd.print(fullLine2.substring(pos2, pos2 + 16));
  } else {
    lcd.print(fullLine2);
  }
}

void scrollIfNeeded() {
  bool needUpdate = false;

  if (fullLine1.length() > 16) {
    pos1++;
    if (pos1 > fullLine1.length()) pos1 = 0;
    needUpdate = true;
  }
  if (fullLine2.length() > 16) {
    pos2++;
    if (pos2 > fullLine2.length()) pos2 = 0;
    needUpdate = true;
  }

  if (needUpdate) updateDisplay();
}

void handleRoot() {
  String html = R"=====(
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>ESP8266 LCD Control</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { background: #000; color: #FF6600; font-family: Arial; text-align: center; padding: 30px; }
    input { width: 90%; padding: 15px; margin: 10px; font-size: 20px; background: #111; color: #B56DF7; border: 1px solid #B56DF7; }
    button { width: 94%; padding: 18px; font-size: 22px; background: #B56DF7; color: #000; border: none; cursor: pointer; }
    .preview { font-family: monospace; font-size: 22px; background: #111; padding: 20px; border: 2px solid #49b8d7; margin-top: 20px; }
    small { color: #0a0; }
  </style>
</head>
<body>
  <h1>Wireless LCD with 'nodeMCU'</h1>
  <input id="l1" placeholder="Line 1">
  <input id="l2" placeholder="Line 2">
  <button onclick="sendText()">UPDATE LCD</button>
  <h2>Output on LCD:-</h2>
  <div class="preview" id="preview">ESP8266 Ready<br>Type message!</div>
  <br><small>By Madhav Vaghamshi</small>

  <script>
    function sendText() {
      let txt1 = document.getElementById('l1').value;
      let txt2 = document.getElementById('l2').value;
      fetch('/set?l1=' + encodeURIComponent(txt1) + '&l2=' + encodeURIComponent(txt2))
        .then(() => {
          let p1 = txt1 || " ";
          let p2 = txt2 || " ";
          document.getElementById('preview').innerHTML = 
            p1.substring(0,40)+(p1.length>40?"...":"") + '<br>' +
            p2.substring(0,40)+(p2.length>40?"...":"");
        });
    }
    document.addEventListener('keypress', e => { if (e.key==='Enter') sendText(); });
  </script>
</body>
</html>
)=====";
  server.send(200, "text/html", html);
}

void handleSet() {
  if (server.hasArg("l1") && server.hasArg("l2")) {
    fullLine1 = server.arg("l1");
    fullLine2 = server.arg("l2");

    // Add spaces at the end for smooth looping
    if (fullLine1.length() > 16) fullLine1 += "    ";
    if (fullLine2.length() > 16) fullLine2 += "    ";

    pos1 = 0;
    pos2 = 0;
    updateDisplay();
  }
  server.send(200, "text/plain", "OK");
}

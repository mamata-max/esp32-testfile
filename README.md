# esp32-testfile
// ESP32  Speed Test: Internal (HTTP) vs External (HTTPS GitHub Raw)
// -------------------------------------------------------------------------
// • Internal  → xxx/20MB.bin  (WiFiClient)
// • External  → https://raw.githubusercontent.com/mamata-max/esp32-testfile/main/20MB.bin  (WiFiClientSecure)
//   - Uses TLS; setInsecure() + setSNI(true) so GitHub Raw accepts the request.
//   - Use correct Host header *raw.githubusercontent.com* and path starting with “/”.
// • Writes to InfluxDB measurement  wifi_speedtest  with tag  site=internal|externals
//   Fields: Download_speed_mbp, Download_time_sec, Payload_bytes,
//           WiFi_rssi_dbm, ping_google_ms, ping_host_ms, Failed_connections

#include "esp_wpa2.h"
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <InfluxDbClient.h>
#include <ESPping.h>

//-------------------------------------------------- CONFIG Wi‑Fi (.1x)
const char* ssid = "xxx";
#define   EAP_IDENTITY   "xxx"
#define   EAP_PASSWORD   "xxx"

//-------------------------------------------------- CONFIG InfluxDB
#define INFLUXDB_URL     "xxx"
#define INFLUXDB_TOKEN   "xxx"
#define INFLUXDB_ORG     "xxx"
#define INFLUXDB_BUCKET  "xxx"
InfluxDBClient influx(INFLUXDB_URL, INFLUXDB_ORG, INFLUXDB_BUCKET, INFLUXDB_TOKEN);

//-------------------------------------------------- Test target struct
struct Target {
  const char* name;        // tag value
  const char* host;        // connect host / ip
  const char* hostHeader;  // HTTP Host header value
  int   port;
  const char* path;        // URL path (starts with /)
  bool  useTLS;            // true → WiFiClientSecure
};

// Internal HTTP (local)
Target internalT = {
  "internal",
  "xxx",
  "xxx",
  xxx,
  "/20MB.bin",
  false
};

// External HTTPS (GitHub Raw)  ← เปลี่ยน username/repo ตามจริง
Target externalT = {
  "external",
  "raw.githubusercontent.com",
  "raw.githubusercontent.com",
  443,
  "/xxx/esp32-testfile/main/20MB.bin",
  true
};

//-------------------------------------------------- Globals
WiFiClient       httpClient;
WiFiClientSecure httpsClient;
unsigned long    lastSafeMillis = 0;
const unsigned long restartInt  = 30UL * 60UL * 1000UL; // 30 min watchdog

struct Result { long bytes = 0; float timeSec = 0; int failed = 1; };

//-------------------------------------------------- download helper
Result runDownload(const Target& T) {
  Result R; WiFiClient* c;
  if (T.useTLS) {
    httpsClient.setInsecure();   // skip root‑CA
    // (ESP32 core ≥2.0 handles SNI automatically when you connect by hostname)
// GitHub requires SNI
    httpsClient.setTimeout(30);
    c = &httpsClient;
  } else {
    httpClient.setTimeout(30);
    c = &httpClient;
  }

  if (!c->connect(T.host, T.port, 7000)) return R; // connect fail

  // Send GET
  c->print(String("GET ") + T.path + " HTTP/1.1\r\n" +
           "Host: " + T.hostHeader + "\r\n" +
           "Connection: close\r\n\r\n");

  // Wait header (10 s)
  unsigned long tWait = millis();
  while (!c->available() && millis() - tWait < 10000) delay(10);

  // Skip header
  while (c->available()) {
    String line = c->readStringUntil('\n');
    if (line == "\r" || line.length() == 0) break;
  }

  uint8_t buf[2048];
  unsigned long t0 = millis();
  while (c->connected() || c->available()) {
    int len = c->read(buf, sizeof(buf));
    if (len > 0) R.bytes += len;
    yield();
  }
  R.timeSec = (millis() - t0) / 1000.0;
  R.failed  = 0;
  c->stop();
  return R;
}

void logResult(const Target& T, const Result& R) {
  float speedMbps = (R.bytes > 0 && R.timeSec > 0) ? (R.bytes * 8.0) / (R.timeSec * 1000000.0) : 0;

  Ping.ping("www.google.com", 4);
  int pingGoogle = Ping.averageTime();
  Ping.ping(T.host, 4);
  int pingHost   = Ping.averageTime();

  Serial.printf("\n[%s] Size: %.2f KB Time: %.2f s Speed: %.3f Mbps  G:%d ms H:%d ms\n",
                T.name, R.bytes / 1024.0, R.timeSec, speedMbps, pingGoogle, pingHost);

  Point p("xxx");
  p.addTag("site", T.name);
  p.addField("Download_speed_mbps", speedMbps);
  p.addField("Download_time_sec",  R.timeSec);
  p.addField("Payload_bytes",      (float)R.bytes);
  p.addField("WiFi_rssi_dbm",      WiFi.RSSI());
  p.addField("ping_google_ms",     pingGoogle);
  p.addField("ping_host_ms",       pingHost);
  p.addField("Failed_connections", R.failed);
  p.addField("ip_address",         WiFi.localIP().toString());
  influx.writePoint(p);
}

//-------------------------------------------------- setup / loop
void connectEduroam() {
  WiFi.mode(WIFI_STA);
  esp_wifi_sta_wpa2_ent_set_identity((uint8_t*)EAP_IDENTITY, strlen(EAP_IDENTITY));
  esp_wifi_sta_wpa2_ent_set_username((uint8_t*)EAP_IDENTITY, strlen(EAP_IDENTITY));
  esp_wifi_sta_wpa2_ent_set_password((uint8_t*)EAP_PASSWORD, strlen(EAP_PASSWORD));
  esp_wifi_sta_wpa2_ent_enable();
  WiFi.begin(ssid);
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) { Serial.print('.'); delay(500);}  
  Serial.printf("\n Wi‑Fi OK IP:%s\n", WiFi.localIP().toString().c_str());
}

void setup() {
  Serial.begin(115200);
  connectEduroam();
  if (influx.validateConnection()) Serial.println("Influx ready");
  lastSafeMillis = millis();
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) ESP.restart();
  if (millis() - lastSafeMillis > restartInt) ESP.restart();

  Result rInt = runDownload(internalT);
  logResult(internalT, rInt);

  Result rExt = runDownload(externalT);
  logResult(externalT, rExt);

  delay(120000);
}


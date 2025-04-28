#define BLYNK_TEMPLATE_ID "Your-Blynk-Template-Id"
#define BLYNK_TEMPLATE_NAME "Your-Blynk-Template-Name"
#define BLYNK_AUTH_TOKEN "Your-Blynk-Auth-Token"

#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>

// --------------------------
// Konfigurasi WiFi
// --------------------------
char auth[] = BLYNK_AUTH_TOKEN;
char ssid[] = "Your-Name-Wifi";
char pass[] = "Your-Pass-Wifi";

// --------------------------
// Konfigurasi Pin
// --------------------------
#define MQ2_PIN      34
#define RELAY_PIN    26
#define LED_ALARM    25
#define BUZZER_PIN   19

const float RL = 10.0;
float Ro = 1.86;
const float batasAsap = 15.0;

bool statusAsap = false;  

void setup() {
  Serial.begin(115200);

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_ALARM, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(LED_ALARM, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  // Cek koneksi WiFi
  WiFi.begin(ssid, pass);
  Serial.println("Mencoba koneksi WiFi...");
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("Terhubung ke WiFi!");

  // Cek koneksi Blynk
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
  if (Blynk.connect()) {
    Serial.println("Terhubung ke Blynk");
  } else {
    Serial.println("Gagal terhubung ke Blynk");
  }

  Serial.println("Sistem monitoring asap siap...");
}

void loop() {
  Blynk.run();

  float ppm = bacaMQ2();

  Serial.print("Kadar Asap: ");
  Serial.print(ppm);
  Serial.println(" ppm");

  // Kirim nilai ppm ke Blynk
  Blynk.virtualWrite(V0, ppm);   // V0 = ppm asap
  Blynk.virtualWrite(V1, statusAsap); // V1 = status alarm

  if (ppm > batasAsap && !statusAsap) {
    Serial.println(">>> Asap TERDETEKSI! Mengaktifkan alarm dan kirim notifikasi...");
    digitalWrite(RELAY_PIN, HIGH);
    digitalWrite(LED_ALARM, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    statusAsap = true;

    // Kirim notifikasi ke HP lewat Blynk
    Blynk.logEvent("asap_terdeteksi", "Asap terdeteksi! Cek lokasi sekarang!");
  }
  else if (ppm <= batasAsap && statusAsap) {
    Serial.println(">>> Aman. Mematikan alarm...");
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(LED_ALARM, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    statusAsap = false;
  }

  Serial.println("----------------------------");
  delay(2000);
}

// --------------------------
// Fungsi baca MQ-2
// --------------------------
float bacaMQ2() {
  float totalRatio = 0;
  for (int i = 0; i < 30; i++) {
    int adcValue = analogRead(MQ2_PIN);
    float voltage = (adcValue / 4095.0) * 3.3;
    if (voltage == 0) voltage = 0.01;
    float Rs = (3.3 - voltage) * RL / voltage;
    float ratio = Rs / Ro;
    totalRatio += ratio;
    delay(20);
  }

  float avgRatio = totalRatio / 30.0;
  float m = -0.42;
  float b = 1.60;
  float ppm = pow(10, ((log10(avgRatio) - b) / m));
  
  // Debug: Cek pembacaan sensor MQ2
  Serial.print("Nilai ADC: ");
  Serial.println(analogRead(MQ2_PIN));
  Serial.print("Voltan MQ2: ");
  Serial.println((analogRead(MQ2_PIN) / 4095.0) * 3.3);
  
  return ppm;
}

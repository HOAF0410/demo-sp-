# demo-sp-
mạch chạy cho esp32 lưu dữ liệu từ cảm biến vào thẻ và có thể truy cập bằng webserver
 mạch không chạy có phát được wifi
// Thêm các thư viện cần thiết cho ESP32
#include <WiFi.h>       // Thư viện này cho phép ESP32 kết nối mạng WiFi
#include <WebServer.h>  // Thư viện để thiết lập một server web đơn giản trên ESP32
#include <SD.h>         // Thư viện hỗ trợ làm việc với thẻ SD
#include <SPI.h>        // Thư viện SPI dùng để kết nối tốc độ cao giữa ESP32 và thẻ SD

// Khai báo các biến và cấu hình phần cứng
const int chipSelect = 5;       // Chân CS (Chip Select) của thẻ SD kết nối với GPIO 5 của ESP32
const int sensorPin = 34;       // Chân kết nối với cảm biến âm thanh (chân GPIO 34 của ESP32)
const int buttonPin = 33;       // Chân kết nối với nút nhấn để bật/tắt ghi âm (chân GPIO 33)
bool recording = false;         // Biến để theo dõi trạng thái ghi âm, ban đầu đặt là không ghi (false)

WebServer server(80);           // Khởi tạo server web trên ESP32, lắng nghe trên cổng 80

// Khởi tạo kết nối WiFi
const char* ssid = "your_SSID";        // Tên mạng WiFi
const char* password = "your_PASSWORD"; // Mật khẩu WiFi

void setup() {
    Serial.begin(115200);               // Khởi động Serial Monitor với tốc độ baud 115200, giúp in dữ liệu ra máy tính để theo dõi

    pinMode(buttonPin, INPUT);          // Thiết lập chân nút nhấn là đầu vào để ESP32 đọc tín hiệu từ nút
    if (!SD.begin(chipSelect)) {        // Kiểm tra nếu thẻ SD không kết nối thành công
        Serial.println("Card failed, or not present");
        return;                         // Nếu thẻ SD không hoạt động, dừng chương trình
    }
    Serial.println("Card initialized.");

    WiFi.begin(ssid, password);         // Kết nối với mạng WiFi
    while (WiFi.status() != WL_CONNECTED) { // Đợi kết nối hoàn thành
        delay(500);
        Serial.print(".");
    }
    Serial.println("\nConnected to WiFi");

    server.on("/", HTTP_GET, []() {     // Xử lý yêu cầu truy cập gốc "/" của web server
        server.send(200, "text/plain", "ESP32 Audio Recorder");
    });

    server.begin();                     // Bắt đầu server web để ESP32 có thể lắng nghe yêu cầu
}

// Hàm bắt đầu ghi âm
void startRecording() {
    File file = SD.open("/recording.wav", FILE_WRITE); // Mở file ghi âm trên thẻ SD
    if (!file) {
        Serial.println("Failed to open file for writing");
        return;
    }
    // Ghi tiêu đề WAV để lưu âm thanh đúng định dạng
    writeWAVHeader(file);
    recording = true;                   // Đặt trạng thái ghi âm thành true để bắt đầu ghi
    Serial.println("Recording started.");
}

// Hàm dừng ghi âm
void stopRecording() {
    recording = false;                  // Đặt trạng thái ghi âm về false để ngừng ghi
    Serial.println("Recording stopped.");
}

// Hàm tạo tiêu đề WAV
void writeWAVHeader(File &file) {
    // Ghi tiêu đề WAV chuẩn gồm thông tin về định dạng và độ dài file
    file.write("RIFF");
    file.write(0); file.write(0); file.write(0); file.write(0); // Kích thước file
    file.write("WAVE");
    file.write("fmt ");
    file.write(16, 4); // Kích thước vùng fmt
    file.write(1, 2);  // Định dạng PCM (không nén)
    file.write(1, 2);  // Số kênh âm thanh (mono)
    file.write(44100, 4); // Tần số lấy mẫu (44100 Hz)
    file.write(88200, 4); // Tốc độ dữ liệu (bitrate) - 44100 * 2 bytes
    file.write(2, 2);  // Số bytes của một khung dữ liệu
    file.write(16, 2); // Độ sâu bit (16-bit)
    file.write("data");
    file.write(0); file.write(0); file.write(0); file.write(0); // Kích thước vùng dữ liệu
}

void loop() {
    if (digitalRead(buttonPin) == HIGH && !recording) { // Nếu nút nhấn được bấm và chưa ghi âm
        startRecording();              // Bắt đầu ghi âm
    } else if (digitalRead(buttonPin) == LOW && recording) { // Nếu nút nhấn không bấm và đang ghi âm
        stopRecording();               // Dừng ghi âm
    }

    if (recording) {                   // Nếu đang ghi âm
        int sensorValue = analogRead(sensorPin); // Đọc giá trị âm thanh từ cảm biến
        sensorValue -= 2048;           // Chuẩn hóa giá trị từ cảm biến về trung tâm (0)
        
        // Ghi dữ liệu cảm biến vào file
        File file = SD.open("/recording.wav", FILE_APPEND);
        file.write(sensorValue & 0xFF);       // Ghi byte thấp
        file.write((sensorValue >> 8) & 0xFF);// Ghi byte cao
        file.close();
    }
    server.handleClient();             // Xử lý yêu cầu của web server
}

// Chú thích:
// - Tín hiệu từ cảm biến được đọc dưới dạng số nguyên từ 0-4095. Việc trừ đi 2048 giúp đưa giá trị này về
//   phạm vi -2048 đến +2047, tương ứng với dạng sóng âm thanh dao động quanh trung tâm 0.
// - Hàm `writeWAVHeader()` tạo tiêu đề cho file WAV với định dạng chuẩn, giúp các phần mềm âm thanh đọc được file. 
// - Web server tại "/" trả về thông báo khi ESP32 được truy cập từ trình duyệt. 

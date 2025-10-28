/*
 * Code Arduino điều khiển máy quét 2 CHIỀU
 * Hardware: Arduino, TB6600, NEMA 17
 * Yêu cầu:
 * 1. Quay 91 bước, mỗi bước dừng 3 giây.
 * 2. Dừng 10 giây ở cuối.
 * 3. Quay ngược 91 bước, mỗi bước dừng 3 giây.
 * 4. Đảm bảo độ chính xác, không sai số tích lũy.
 */

// --- 1. CÀI ĐẶT PHẦN CỨNG ---
#define DIR_PIN 2
#define STEP_PIN 3
#define ENA_PIN 4 

// --- 2. THÔNG SỐ ĐỘNG CƠ VÀ DRIVER (PHẢI TRÙNG KHỚP THỰC TẾ) ---
const int MOTOR_STEPS_PER_REV = 200;  // 200 bước/vòng (1.8 độ)
const int MICROSTEPPING = 16;         // Đang dùng 1/16 vi bước

// --- 3. THÔNG SỐ QUÁ TRÌNH QUÉT (THEO YÊU CẦU) ---
const int PULSE_DELAY = 500;            // 500us (tốc độ quay)
const long MEASUREMENT_DELAY_NORMAL = 3000; // 3 giây
const long MEASUREMENT_DELAY_END = 10000;   // 10 giây

// --- 4. CÁC TRẠNG THÁI CỦA MÁY ---
#define STATE_SCAN_FORWARD 0  // Đang quét tới
#define STATE_END_PAUSE 1     // Đang dừng 10s ở cuối
#define STATE_SCAN_REVERSE 2  // Đang quét lùi
#define STATE_FINISHED 3      // Đã hoàn thành

int currentState = STATE_SCAN_FORWARD; // Trạng thái ban đầu
int currentStop = 0;                   // Vị trí hiện tại (0 = 190nm, 91 = 1100nm)

// --- 5. TÍNH TOÁN ĐỘ CHÍNH XÁC (TỰ ĐỘNG) ---
const long TOTAL_MICROSTEPS = (long)MOTOR_STEPS_PER_REV * MICROSTEPPING; // 3200
const int NUM_STOPS = 91;
const int BASE_STEPS = TOTAL_MICROSTEPS / NUM_STOPS;     // 3200 / 91 = 35
const int REMAINDER_STEPS = TOTAL_MICROSTEPS % NUM_STOPS; // 3200 % 91 = 15
long error_accumulator = 0; // Biến tích lũy lỗi

// --- 6. CHƯƠNG TRÌNH CHÍNH ---

void setup() {
  Serial.begin(9600);
  Serial.println("Khoi dong chu trinh quet 2 chieu...");

  pinMode(STEP_PIN, OUTPUT);
  pinMode(DIR_PIN, OUTPUT);
  pinMode(ENA_PIN, OUTPUT);

  // Kích hoạt driver (LOW = Enable)
  digitalWrite(ENA_PIN, LOW);
  
  Serial.print("Tong vi buoc: "); Serial.println(TOTAL_MICROSTEPS);
  Serial.print("So diem dung: "); Serial.println(NUM_STOPS);
  Serial.print("Buoc co ban (35): "); Serial.println(NUM_STOPS - REMAINDER_STEPS);
  Serial.print("Buoc bu (36): "); Serial.println(REMAINDER_STEPS);
}

void loop() {
  
  switch (currentState) {
    
    // --- TRẠNG THÁI 1: QUÉT TỚI ---
    case STATE_SCAN_FORWARD:
      if (currentStop < NUM_STOPS) {
        
        Serial.print("Tien -> Diem dung #");
        Serial.print(currentStop + 1); // Đếm từ 1 đến 91
        
        digitalWrite(DIR_PIN, HIGH); // Đặt chiều quay TỚI
        moveOneAccurateStep();       // Quay 1 bước (35 hoặc 36 vi bước)
        
        currentStop++; // Cập nhật vị trí
        
        Serial.println("... Dung 3 giay.");
        delay(MEASUREMENT_DELAY_NORMAL); // Dừng 3 giây
        
      } else {
        // Đã đến điểm 91 (1100nm)
        currentState = STATE_END_PAUSE; // Chuyển trạng thái
        Serial.println("--- DA DEN 1100nm. DUNG 10 GIAY ---");
        delay(MEASUREMENT_DELAY_END); // Dừng 10 giây
      }
      break;

    // --- TRẠNG THÁI 2: DỪNG 10S (Tự chuyển) ---
    case STATE_END_PAUSE:
      // Sau khi dừng 10s xong, chuyển sang quay lùi
      currentState = STATE_SCAN_REVERSE;
      Serial.println("Bat dau quay nguoc ve 190nm...");
      
      // Reset biến lỗi về 0 (dù nó đã = 0 nhưng để cho rõ ràng)
      error_accumulator = 0; 
      break;

    // --- TRẠNG THÁI 3: QUÉT LÙI ---
    case STATE_SCAN_REVERSE:
      if (currentStop > 0) {
        
        Serial.print("Lui <- Diem dung #");
        Serial.print(currentStop); // Đếm từ 91 về 1
        
        digitalWrite(DIR_PIN, LOW); // Đặt chiều quay LÙI
        moveOneAccurateStep();      // Quay 1 bước (35 hoặc 36 vi bước)
        
        currentStop--; // Cập nhật vị trí
        
        Serial.println("... Dung 3 giay.");
        delay(MEASUREMENT_DELAY_NORMAL); // Dừng 3 giây
        
      } else {
        // Đã về đến điểm 0 (190nm)
        currentState = STATE_FINISHED; // Chuyển trạng thái
      }
      break;

    // --- TRẠNG THÁI 4: KẾT THÚC ---
    case STATE_FINISHED:
      Serial.println("--- HOAN THANH CHU TRINH ---");
      digitalWrite(ENA_PIN, HIGH); // Tắt động cơ (Disable)
      while (1); // Dừng chương trình vĩnh viễn
      break;
  }
}

/**
 * @brief Hàm thực hiện MỘT bước nhảy (35 hoặc 36 vi bước)
 * Sử dụng thuật toán phân bổ lỗi để đảm bảo độ chính xác.
 */
void moveOneAccurateStep() {
  // 1. Tính số bước cần quay cho lần này
  int steps_this_move = BASE_STEPS; // Bắt đầu bằng bước cơ bản (35)
  
  // 2. Cộng dồn phần dư (lỗi)
  error_accumulator += REMAINDER_STEPS; // Cộng 15
  
  // 3. Kiểm tra xem lỗi đã đủ "tràn" chưa (>= 91)
  if (error_accumulator >= NUM_STOPS) {
    steps_this_move++; // Nếu tràn, lần này ta quay 36 bước
    error_accumulator -= NUM_STOPS; // và trừ lỗi đi (vd: 105 - 91 = 14)
  }
  
  Serial.print(" (Quay ");
  Serial.print(steps_this_move);
  Serial.print(" vi buoc)");

  // 4. Thực hiện quay
  moveMicrosteps(steps_this_move);
}

/**
 * @brief Hàm phát xung để quay động cơ
 * @param steps Số vi bước cần quay
 */
void moveMicrosteps(int steps) {
  for (int i = 0; i < steps; i++) {
    digitalWrite(STEP_PIN, HIGH);
    delayMicroseconds(5); 
    digitalWrite(STEP_PIN, LOW);
    delayMicroseconds(PULSE_DELAY);
  }
}

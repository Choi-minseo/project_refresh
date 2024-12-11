# project_refresh

#include <SoftwareSerial.h>

SoftwareSerial mySerial(10, 11); // RX, TX 핀 설정

void setup() {
  Serial.begin(9600); // 기본 시리얼 모니터
  mySerial.begin(9600); // CM1106과 통신용 시리얼
  
  Serial.println("CM1106 CO2 Sensor Test");
}

void loop() {
  byte request[] = {0x11, 0x01, 0x01, 0xED}; // CM1106 데이터 요청 명령
  byte response[8]; // 응답 저장용 배열

  mySerial.write(request, sizeof(request)); // 센서에 데이터 요청
  delay(100); // 센서 응답 대기
  
  if (mySerial.available() >= 8) { // 응답 데이터가 충분히 도착했는지 확인
    for (int i = 0; i < 8; i++) {
      response[i] = mySerial.read(); // 데이터 읽기
    }

    // 응답 데이터에서 CO2 농도 추출 (데이터 시트 참고)
    if (response[0] == 0x16 && response[1] == 0x02) {
      int co2 = (response[2] << 8) | response[3];
      Serial.println("CO2: " + String(co2) + " ppm");
    }
  }
  delay(2000); // 2초마다 데이터 요청
}


아두이노(Arduino)와 젯슨 나노(Jetson Nano)를 사용하여 온도, 습도, CO2 데이터를 수집하고 90분 동안 데이터를 엑셀 파일로 저장하는 작업을 설정하는 방법을 단계적으로 설명하겠습니다.

단계 1: 아두이노 코드 작성 및 업로드

1.1. 아두이노에서 데이터를 전송하도록 설정
아두이노는 데이터를 쉼표로 구분된 형식(예: 25.3,60.5,400)으로 젯슨 나노에 전송해야 합니다. 아래는 아두이노 코드 예제입니다:

#include "DHT.h"

#define DHTPIN 2     // DHT22 센서 핀
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(9600); // 시리얼 통신 시작
  dht.begin();
}

void loop() {
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();
  int co2 = analogRead(A0); // CO2 센서 데이터를 읽기 (예시)

  if (isnan(temperature) || isnan(humidity)) {
    Serial.println("Error reading DHT sensor data!");
    delay(2000);
    return;
  }

  // 쉼표로 구분된 데이터 전송
  Serial.print(temperature);
  Serial.print(",");
  Serial.print(humidity);
  Serial.print(",");
  Serial.println(co2);

  delay(2000); // 2초 간격으로 데이터 전송
}

코드 업로드:

아두이노 IDE에서 위 코드를 작성한 후 아두이노에 업로드합니다.
업로드 후 실행:

업로드가 완료되면 아두이노는 자동으로 데이터를 전송합니다.
이후 아두이노 IDE를 종료해 시리얼 포트가 해제되도록 합니다. (중복 포트 사용 방지)

단계 2: 젯슨 나노에서 Python 코드 작성
아두이노로부터 데이터를 수신하여 90분 동안 수집하고, 이를 엑셀 파일로 저장하는 Python 코드를 작성합니다.
import serial
import pandas as pd
import time

# 시리얼 포트 설정 (Jetson Nano의 UART 포트 사용)
try:
    sensor_port = serial.Serial('/dev/ttyUSB0', 9600, timeout=2)  # Arduino 연결 포트
except serial.SerialException as e:
    print(f"Error: Could not open port. {e}")
    exit(1)

data_list = []

# 데이터 수집 및 저장 함수
def collect_and_save_data(duration=90):  # 기본값: 90분 동안 실행
    start_time = time.time()
    formatted_start_time = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(start_time))

    try:
        while time.time() - start_time < duration * 60:  # 정해진 시간 동안 실행
            if sensor_port.in_waiting > 0:
                raw_data = sensor_port.readline().decode('utf-8').strip()
                print(f"Raw data received: {raw_data}")
                try:
                    # 데이터가 쉼표로 구분된 3개의 값인지 확인
                    if len(raw_data.split(",")) == 3:
                        temperature, humidity, co2 = raw_data.split(",")
                        timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
                        data_list.append({
                            "Timestamp": timestamp,
                            "Temperature": float(temperature),
                            "Humidity": float(humidity),
                            "CO2_Level": float(co2)
                        })
                        print(f"Parsed: Temperature={temperature}, Humidity={humidity}, CO2={co2}")
                    else:
                        print(f"Invalid data format: {raw_data}")
                except ValueError as e:
                    print(f"Parsing error: {raw_data}. Error: {e}")
            else:
                print("No data received from sensor. Waiting...")

        # 시작 시간과 종료 시간 추가
        formatted_end_time = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(time.time()))
        data_list.append({"Timestamp": "Start Time", "Temperature": "-", "Humidity": "-", "CO2_Level": formatted_start_time})
        data_list.append({"Timestamp": "End Time", "Temperature": "-", "Humidity": "-", "CO2_Level": formatted_end_time})

        # 데이터프레임 생성 및 엑셀 저장
        df = pd.DataFrame(data_list)
        print(df.head())  # 데이터프레임 출력 (확인용)
        file_path = "/home/dli/sensor_data.xlsx"  # 저장 경로
        df.to_excel(file_path, index=False)
        print(f"Excel file saved as '{file_path}'")

    except serial.SerialException as e:
        print(f"Serial exception: {e}")
    except KeyboardInterrupt:
        print("Data collection stopped by user.")

    finally:
        if sensor_port.is_open:
            sensor_port.close()

# 프로그램 실행
if __name__ == "__main__":
    collect_and_save_data(duration=90)  # 90분 동안 데이터 수집

Python 코드 저장:

이 코드를 save1_data.py라는 이름으로 저장합니다:
bash
코드 복사
nano /home/dli/save1_data.py
저장 후 종료: Ctrl + O, Enter, Ctrl + X


단계 3: 자동 실행 설정 (선택 사항)
Jetson Nano가 전원만 연결되면 코드를 자동 실행하도록 설정하려면, systemd 서비스를 설정합니다.

1.서비스 파일 생성:

sudo nano /etc/systemd/system/save1_data.service

2.다음 내용을 입력:

[Unit]
Description=Sensor Data Collector Service
After=multi-user.target

[Service]
ExecStart=/usr/bin/python3 /home/dli/save1_data.py
Restart=always
User=dli

[Install]
WantedBy=multi-user.target

3.저장 후 종료: Ctrl + O, Enter, Ctrl + X

4.서비스 활성화:

sudo systemctl enable save1_data.service
sudo systemctl start save1_data.service
sudo systemctl status save1_data.service >> 활성화 상태 확인 코드

단계 4: 저장된 엑셀파일을 이메일로 전송
저장된 엑셀 파일을 이메일로 전송하려면 Python을 사용해 Gmail SMTP를 활용하는 방법이 일반적입니다. 아래는 Python 코드를 작성하는 방법입니다.


python코드를 send_email.py로 저장
nano send_email.py


코드: 엑셀 파일을 이메일로 전송
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders
import os

def send_email(file_path, sender_email, receiver_email, password):
    """
    엑셀 파일을 이메일로 전송하는 함수.
    
    Parameters:
        file_path (str): 첨부할 엑셀 파일 경로
        sender_email (str): 발신자의 Gmail 주소
        receiver_email (str): 수신자의 이메일 주소
        password (str): Gmail 앱 비밀번호
    """
    # 파일 존재 여부 확인
    if not os.path.exists(file_path):
        print(f"Error: File '{file_path}' does not exist.")
        return

    try:
        # 이메일 메시지 설정
        msg = MIMEMultipart()
        msg['From'] = sender_email
        msg['To'] = receiver_email
        msg['Subject'] = "Sensor Data (Excel File Attached)"

        # 첨부 파일 설정
        attachment = MIMEBase('application', 'octet-stream')
        with open(file_path, 'rb') as file:
            attachment.set_payload(file.read())
        encoders.encode_base64(attachment)
        attachment.add_header('Content-Disposition', f'attachment; filename={os.path.basename(file_path)}')
        msg.attach(attachment)

        # Gmail SMTP 서버에 연결
        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
            server.login(sender_email, password)
            server.sendmail(sender_email, receiver_email, msg.as_string())

        print(f"Email sent successfully to {receiver_email} with file '{file_path}' attached.")

    except Exception as e:
        print(f"Failed to send email: {e}")

# 실행 예제
if __name__ == "__main__":
    # 엑셀 파일 경로와 이메일 설정
    file_path = "/home/your_username/sensor_data.xlsx"  # 저장된 엑셀 파일 경로
    sender_email = "your_email@gmail.com"              # 발신자의 Gmail 주소
    receiver_email = "recipient_email@gmail.com"       # 수신자의 이메일 주소
    password = "your_app_password"                     # Gmail 앱 비밀번호

    # 이메일 전송 함수 호출
    send_email(file_path, sender_email, receiver_email, password)



이메일 전송 실행 
python3 send_email.py













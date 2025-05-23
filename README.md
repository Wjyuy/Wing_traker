## 비행체 추적 장치 (Wing Tracker)

### 프로젝트 개요

**목표:** 
임베디드장치와 코딩을 활용하여 비행체를 실시간으로 인식하고 추적하는 시스템을 구축
**프로젝트 배경:** 불법 드론이나 레이더에 포착되지 않는 오물 풍선 등으로 인한 피해 및 시민의 불안감 증가에 대한 문제의식에서 시작

### 주요 목표

* **비행체 인식 시스템 구현:** YOLO 알고리즘을 활용하여 카메라 영상에서 비행체를 실시간으로 인식하고 그 위치를 추적하는 시스템을 구축
* **추적 정확도 향상:** YOLO모델을 직접 학습시키고, ROBOFLOW의 데이터에 추가적인 학습을 통해 비행체 추적의 정확도를 향상
* **시제품 제작 및 테스트:** 시스템의 성능을 평가하고 비행체 추적 정확도를 검증하는 임베디드 장치를 제작하고 테스트를 수행

### 기술 개요 및 시스템 설계

#### 객체 탐지 알고리즘: YOLO (You Only Look Once)

YOLO는 이미지나 비디오에서 객체를 실시간으로 인식하고 위치를 파악하는 객체 탐지 알고리즘

* **실시간 객체 탐지:** 
* 단일 신경망을 사용하여 이미지를 한 번에 분석하고 객체를 인식하는 방식을 사용
* 객체 탐지를 회귀 문제로 재구성하여 공간적으로 분리된 경계 상자와 관련 클래스 확률을 직접 예측
* 전체 탐지 파이프라인이 단일 네트워크이기 때문에 탐지 성능에 대해 종단간(end-to-end) 최적화가 가능

* **빠른 처리 속도:** 
* 이미지를 한 번만 처리하여 객체를 탐지하므로 실시간 처리가 가능
* 일부 버전에서는 초당 45프레임 이상을 처리
* YOLO 논문의 기본 모델(Base YOLO)은 초당 45프레임, Fast YOLO는 초당 155프레임으로 이미지를 처리

* **단순하고 효율적인 구조:** Faster R-CNN과 같은 다단계 탐지기보다 간단하고 효율적
* YOLO 논문에서는 R-CNN과 DPM 같은 기존 방식이 분류기를 재활용하거나 복잡한 파이프라인을 사용한다고 설명하며, YOLO는 이를 단일 신경망으로 대체하여 단순하고 빠르다고 언급
* **높은 배경 인식 정확도:** 
* 전체 이미지 맥락을 학습하여 배경 탐지 오류를 줄이고 정확도를 높임
*YOLO는 전체 이미지를 보기 때문에 배경에서 잘못된 탐지(false positives)를 예측할 가능성이 적음

#### 사용 모델: YOLOv11n

<table>
  <tr>
    <td><img src="imgs/msing.png" alt="yolo_ms/img"></td>
  </tr>
</table>

* **선택 이유:** 본 프로젝트에서는 라즈베리 파이를 이용
* YOLOv11n은 성능이 낮은 장치에서도 실시간 추적이 가능할 정도로 처리 속도가 빠름
* 모델이 가벼워 각 프레임에 대한 처리 시간이 짧고 ms/img가 적다
* YOLO 모델의 경량화 버전으로 모델 크기가 작아 메모리 사용량도 적다

#### 하드웨어 구성

<table>
  <tr>
    <td><img src="imgs/hardware.png" alt="하드웨어 구성"></td>
  </tr>
</table>

* **메인 컨트롤러:** Raspberry Pi (프로젝트 초반 Jetson Nano 고려, 안정성 문제로 변경).
* **카메라:** IMX 219.
* **서보 모터 제어:** 
* U2D2 (USB to Dynamixel adapter)
* U2D2 hub
* AX12 서보모터 *2 

#### 서보 모터 제어 방식

1.  YOLO 모델을 활용하여 객체를 실시간으로 감지하고 감지된 객체의 좌표 ($\(x_1, y_1, x_2, y_2\)$) 계산
2.  객체의 중심 좌표 ($\(x_{center}, y_{center}\)$)를 계산하여 서보 모터가 이동해야 할 목표 위치를 결정
    * $\(x_{center} = (x_1 + x_2) / 2\)$
    * $\(y_{center} = (y_1 + y_2) / 2\)$
3.  화면 좌표를 서보 모터의 목표 위치 범위 (DXL\_MIN\_POSITION ~ DXL\_MAX\_POSITION)로 매핑
4.  서보 모터의 ID를 사용하여 X축과 Y축 서보 모터를 개별적으로 제어하며, 목표 위치로 부드럽게 이동하도록 구현
5.  계산된 목표 위치가 지정된 범위를 벗어나지 않도록 유효성 검사 및 보정을 수행
6.  실시간으로 객체를 추적하고, 객체가 화면에 나타날 때마다 서보 모터를 해당 위치로 이동시켜 물리적 공간에서 객체를 추적

```python
    from ultralytics import YOLO
    from dynamixel_sdk import *  # Dynamixel SDK library
    import time

    # Dynamixel configuration
    ADDR_TORQUE_ENABLE = 24
    ADDR_GOAL_POSITION = 30
    ADDR_PRESENT_POSITION = 36

    PROTOCOL_VERSION = 1.0
    DXL_ID_X = 2  # Dynamixel ID for X-axis
    DXL_ID_Y = 1  # Dynamixel ID for Y-axis
    BAUDRATE = 1000000
    DEVICENAME = '/dev/ttyUSB0'

    TORQUE_ENABLE = 1
    DXL_MIN_POSITION = 600  # Minimum position value
    DXL_MAX_POSITION = 1000  # Maximum position value

    SCREEN_WIDTH = 1200
    SCREEN_HEIGHT = 1200

    DEAD_ZONE = 20  # Dead zone in pixels
    SMOOTH_STEP = 2  # Movement step size for smooth motion
    DELAY_BETWEEN_MOVES = 0.1 # Delay between servo movements (in seconds)

    # Initialize PortHandler and PacketHandler
    port_handler = PortHandler(DEVICENAME)
    packet_handler = PacketHandler(PROTOCOL_VERSION)

    # Open port
    if not port_handler.openPort():
        print("Failed to open port")
        exit()

    # Set baudrate
    if not port_handler.setBaudRate(BAUDRATE):
        print("Failed to set baudrate")
        exit()

    # Enable torque for both servos
    for DXL_ID in [DXL_ID_X, DXL_ID_Y]:
        dxl_comm_result, dxl_error = packet_handler.write1ByteTxRx(port_handler, DXL_ID, ADDR_TORQUE_ENABLE, TORQUE_ENABLE)
        if dxl_comm_result != COMM_SUCCESS:
            print(f"Communication Error: {packet_handler.getTxRxResult(dxl_comm_result)}")
        elif dxl_error != 0:
            print(f"Torque Enable Error for ID {DXL_ID}: {packet_handler.getRxPacketError(dxl_error)}")
        else:
            print(f"Dynamixel ID {DXL_ID} successfully connected")

    # Function to gradually move servo to the target position
    def move_servo_smoothly(packet_handler, port_handler, dxl_id, current_position, target_position, step=SMOOTH_STEP):
        while abs(current_position - target_position) > step:
            if current_position < target_position:
                current_position += step
            elif current_position > target_position:
                current_position -= step
            
            # Write intermediate position
            dxl_comm_result, dxl_error = packet_handler.write2ByteTxRx(port_handler, dxl_id, ADDR_GOAL_POSITION, current_position)
            if dxl_comm_result != COMM_SUCCESS:
                print(f"Communication Error: {packet_handler.getTxRxResult(dxl_comm_result)}")
            elif dxl_error != 0:
                print(f"Dynamixel Error: {packet_handler.getRxPacketError(dxl_error)}")
            
            # Pause slightly to allow smooth movement
            time.sleep(0.01)

        # Set final position
        dxl_comm_result, dxl_error = packet_handler.write2ByteTxRx(port_handler, dxl_id, ADDR_GOAL_POSITION, target_position)
        if dxl_comm_result != COMM_SUCCESS:
            print(f"Final Position Communication Error: {packet_handler.getTxRxResult(dxl_comm_result)}")
        elif dxl_error != 0:
            print(f"Final Position Dynamixel Error: {packet_handler.getRxPacketError(dxl_error)}")

    # YOLO model setup
    model = YOLO('/home/wtf/yolo11n version best weight.pt')
    results = model.predict(source='tcp://127.0.0.1:8888', stream=True, show=True)

    # Initialize current positions for both servos
    current_position_x = 800  # Initial midpoint
    current_position_y = 800

    # Offsets for fine-tuning
    X_OFFSET = 50
    Y_OFFSET = 50

    # Main loop for real-time object tracking
    # Main loop for real-time object tracking
    for r in results:  # Frame-by-frame processing
        for box in r.boxes:
            # Check if the detected object is a person (class ID = 0)
            if box.cls[0] == 0:  # Assuming '0' is the class ID for 'person'
                # Extract center coordinates
                x1, y1, x2, y2 = box.xyxy[0].tolist()
                x_center = (x1 + x2) / 2
                y_center = (y1 + y2) / 2

                # Reverse X and Y mappings
                target_position_x = int(current_position_x + (DXL_MAX_POSITION - DXL_MIN_POSITION) * 0.5*(1-x_center / SCREEN_WIDTH))
                target_position_y = int(current_position_y + (DXL_MAX_POSITION - DXL_MIN_POSITION) * 0.5*(y_center / SCREEN_HEIGHT))

                # Ensure target positions are within valid range
                target_position_x = max(DXL_MIN_POSITION, min(DXL_MAX_POSITION, target_position_x))
                target_position_y = max(DXL_MIN_POSITION, min(DXL_MAX_POSITION, target_position_y))

                # Write directly to servos
                dxl_comm_result_x, dxl_error_x = packet_handler.write2ByteTxRx(port_handler, DXL_ID_X, ADDR_GOAL_POSITION, target_position_x)
                if dxl_comm_result_x != COMM_SUCCESS:
                    print(f"Error moving X servo: {packet_handler.getTxRxResult(dxl_comm_result_x)}")
                elif dxl_error_x != 0:
                    print(f"Dynamixel X Error: {packet_handler.getRxPacketError(dxl_error_x)}")

                dxl_comm_result_y, dxl_error_y = packet_handler.write2ByteTxRx(port_handler, DXL_ID_Y, ADDR_GOAL_POSITION, target_position_y)
                if dxl_comm_result_y != COMM_SUCCESS:
                    print(f"Error moving Y servo: {packet_handler.getTxRxResult(dxl_comm_result_y)}")
                elif dxl_error_y != 0:
                    print(f"Dynamixel Y Error: {packet_handler.getRxPacketError(dxl_error_y)}")

                print(f"Person detected. Moving to (X: {target_position_x}, Y: {target_position_y})")

                # Add delay to slow down updates
                time.sleep(DELAY_BETWEEN_MOVES)


    # Close port when done
    port_handler.closePort()
```

### 시제품 개발 과정 하이라이트

* 설계 및 프로토타입 제작.
* 데이터 전처리 및 라벨링 (labelImg 사용, Balloon, UAV, Drone 클래스).
* YOLO 학습 코드 개발.
* 이미지 필터링(Sobel 등)을 적용한 학습 성능 비교 실험 (Original 데이터 학습과 성능 유사하여 미채용).
* Raspberry Pi 카메라 연결 및 외형 제작.
* 서보 모터 제어 구현.

<table>
  <tr>
    <td><img src="imgs/flow.png" alt="블록 다이어그램"></td>
  </tr>
</table>

### 외형제작
cad를 통한 디자인 진행
<table>
  <tr>
    <td>윗면</td>
    <td>옆면</td>
    <td>측면</td>
  </tr>
  <tr>
    <td><img src="imgs/cad (1).jpg" alt="윗면" width="200"></td>
    <td><img src="imgs/cad (2).jpg" alt="옆면" width="200"></td>
    <td><img src="imgs/cad (3).jpg" alt="측면" width="200"></td>
  </tr>
</table>

### 데이터 전처리 과정

* roboflow 의 데이터 (비행체)
* 추가적인 데이터 전처리 및 라벨링 ([labelImg](https://github.com/HumanSignal/labelImg) 사용, Balloon, UAV, Drone 클래스).

### 데이터 증강 기법

* 오물풍선의 데이터 수집을 위해 좌우반전,밝기조절, 회전, 노이즈, 확대 축소 등을 이용
<table>
  <tr>
    <td>증강 적용 된 오물풍선 이미지들</td>
    <td>labelImg로 크롤링</td>
  </tr>
  <tr>
    <td><img src="imgs/ballons.png" alt="ballons"></td>
    <td><img src="imgs/lab2.png" alt="lab2"></td>
  </tr>
</table>

### 필터 적용 후 학습 

* sobel,hfp,prewitt,scharr,canny 필터 적용
* 예시 : sobel 필터

```python
    import cv2
    import os
    import glob
    import shutil

    # Canny 필터를 적용하고 결과를 저장하는 함수
    def apply_canny(image_path, output_path):
        # 이미지 읽기
        image = cv2.imread(image_path, cv2.IMREAD_GRAYSCALE)

        # Canny 에지 검출 적용
        edges = cv2.Canny(image, 100, 200)  # 첫 번째 인자는 하이Threshold, 두 번째는 로우Threshold

        # 필터링된 이미지 저장 (원래 이름 그대로 저장)
        cv2.imwrite(output_path, edges)

    # 라벨 파일 복사 함수
    def copy_label_file(image_file, input_label_dir, output_label_dir):
        # 이미지 파일명에서 확장자를 .txt로 바꿈
        image_name = os.path.basename(image_file)
        label_name = os.path.splitext(image_name)[0] + '.txt'

        input_label_path = os.path.join(input_label_dir, label_name)
        output_label_path = os.path.join(output_label_dir, label_name)

        # 라벨 파일이 존재할 경우 복사
        if os.path.exists(input_label_path):
            shutil.copy(input_label_path, output_label_path)

    # 디렉토리 내 모든 이미지에 Canny 필터 적용하고 레이블 파일도 복사하는 함수
    def process_images_and_labels(input_image_dir, input_label_dir, output_image_dir, output_label_dir):
        # 결과 저장 경로가 없으면 생성
        os.makedirs(output_image_dir, exist_ok=True)
        os.makedirs(output_label_dir, exist_ok=True)

        # 이미지 파일 처리
        for image_file in glob.glob(os.path.join(input_image_dir, '*.jpg')):  # jpg 형식으로 가정
            image_name = os.path.basename(image_file)
            output_image_path = os.path.join(output_image_dir, image_name)  # _filter 없이 저장

            # Canny 필터 적용 및 저장
            apply_canny(image_file, output_image_path)

            # 라벨 파일 복사
            copy_label_file(image_file, input_label_dir, output_label_dir)

    # 경로 설정
    train_image_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/train/images'
    val_image_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/valid/images'
    test_image_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/test/images'

    train_label_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/train/labels'
    val_label_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/valid/labels'
    test_label_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/test/labels'

    train_output_image_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/train_canny/images'
    val_output_image_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/valid_canny/images'
    test_output_image_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/test_canny/images'

    train_output_label_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/train_canny/labels'
    val_output_label_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/valid_canny/labels'
    test_output_label_dir = '/content/trash-laden-balloons,-UAV,-Drone-1/test_canny/labels'

    # 각 폴더에 대해 Canny 필터 적용 및 라벨 파일 복사
    process_images_and_labels(train_image_dir, train_label_dir, train_output_image_dir, train_output_label_dir)
    process_images_and_labels(val_image_dir, val_label_dir, val_output_image_dir, val_output_label_dir)
    process_images_and_labels(test_image_dir, test_label_dir, test_output_image_dir, test_output_label_dir)

    print("Canny edge detection and label copying completed for train, valid, and test sets.")
```

* 불규칙한 도시지형에서 좀더 쉽게 edge를 감지하고자 edge detection을 위해 다섯 종류의 필터를 적용하여 학습

<table>
  <tr>
    <td>적용 전</td>
    <td>적용 후</td>
  </tr>
  <tr>
    <td><img src="imgs/before.png" alt="b" ></td>
    <td><img src="imgs/after.png" alt="b" ></td>
  </tr>
</table>

* 결과적으로는 Original vesion과 성능이 비슷해 채용하지 않음

<table>
  <tr>
    <td>Training Metrics and Loss</td>
  </tr>
  <tr>
    <td><img src="imgs/ed.png" alt="Training Metrics and Loss" ></td>
  </tr>
</table>

<table>
  <tr>
    <td>Confusion Matrix</td>
  </tr>
  <tr>
    <td><img src="imgs/conf.png" alt="Confusion Matrix" ></td>
  </tr>
</table>

### 결과

* 오물풍선은 학습 결과가 만족스럽진 않았지만, 인식은 되었다
* 무인기는 roboflow 학습 데이터를 사용하여 인식이 잘 되었다 
<table>
  <tr>
    <td>오물풍선</td>
    <td>무인기</td>
  </tr>
  <tr>
    <td><img src="imgs/fin1.png" alt="오물풍선" ></td>
    <td><img src="imgs/fin2.png" alt="무인기" ></td>
  </tr>
</table>


* **주요 문제점:** 라즈베리 파이 사용으로 인해 초당 1프레임대 성능밖에 구현하지 못함

### 문제점 및 개선사항
* opencv 등 다양한 라이브러리 설치시 버전 오류 문제로 인해 발생할 때 마다 초기화 작업이 필요해 진행이 지연되는 문제가 있었다. 이는 가상환경 virtualenv을 사용, 구축하여 해결하였다.

* jetson nano 보드 사용중 보드의 short가 발생하여 AS를 보냈으나 다시 보드를 받기까지 오래 걸려 급히 라즈베리 파이 보드로 대체하여 진행하였다. 또한 라즈베리 파이의 성능 제약을 해결하기 위해 경량화된 YOLO 모델을 적용하여 프레임 처리 속도를 개선하였다. 경량화된 모델을 사용했기 때문에 기존 모델 기준 정확도가 0.9에서 0.5로 감소해 비행체를 인식하는 성능이 떨어져 아쉬웠다.

* 오물 풍선, 드론, 무인기 등의 특수한 객체를 탐지하는데 필요한 이미지를 수집하는데 어려움을 겪었다. 모델 훈련에는 다양한 각도와 환경에서 촬영된 이미지가 필요했는데, 데이터셋을 구성하는데 한계가 있었다. 이는 데이터 증강 기술을 적극적으로 활용해 충분한 양의 학습 데이터를 제공할 수 있었다.

* 라즈베리 파이를 써서 제작해 초당 1프레임 대 정도의 성능밖에 구현하지 못했다.

* 초당 1프레임 객체 인식과 모터 움직임간의 딜레이가 발생해서, 모터 움직임을 딜레이를 줘서 해결하려 시도하였다. 서보모터 제어 파트(16p) def move_servo_smoothly 함수를 이용하여 해결하였다.

* 카메라의 붉은색 색조현상이 발생하였다. 이를 해결하기 위해 필터 색조값을 일일히 수정하려 하였으나 ISP파라미터를 적용하여 조정하였다.

### 향후 계획

* 레이저 거리 측정기를 추가하여 물체의 정확한 거리와 위치를 파악하고, 이를 통해 좀 더 세밀한 추적 장치를 개발할 계획

* Jetson nano 보드의 AS가 완료된다면, 그를 활용하여 기존 Raspberry pi 기반 시스템에서 발생했던 성능 제약을 극복하고 더욱 강력한 YOLO 모델을 적용하여 실시간으로 처리할 수 있는 추적 장치를 개발할 계획이다. 

### 프로젝트 의의와 최종 성과

* 소형 임베디드 보드를 활용하여 딥러닝 기반의 실시간 객체 추적 시스템을 구현. 저비용, 고효율의 IoT 및 AI 융합 기술 개발의 가능성을 보여주는 사례
* 다양한 비행체를 실시간으로 추적하는 기술을 바탕으로 보안, 물류, 탐사, 환경 모니터링 등 다양한 분야에서의 실질적인 활용 가능성 제시
* 드론, 무인기, 투척 물체 등 다양한 비행체를 정확하게 식별, 추적 가능한 객체 추적 장치 개발
* 하드웨어에서의 성능제약을 해결하는데 최적화된 소프트웨어를 결합해 소형화된 형태의 저비용 솔루션을 구현
* 다양한 임베디드 환경에 적용 가능한 경량 시스템 제작 


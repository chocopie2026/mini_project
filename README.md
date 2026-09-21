# mini_project
# (짭) 거짓말 탐지기 (누가 만들었어!)
# 만든 날짜 : 2026.09.21
from machine import Pin, ADC
from neopixel import NeoPixel
import time

# -------------------------------------------------------------
# 1. Pico 2W 전용 NeoPixel & ADC 설정
# -------------------------------------------------------------
adc = ADC(26) # MQ-2 센서 (GP26 / ADC0)

LED_PIN = 16
LED_COUNT = 10
TIMING = (280, 515, 515, 745)

np = NeoPixel(Pin(LED_PIN), LED_COUNT, timing=TIMING)

COLOR_GREEN = (0, 50, 0)    # 1~5칸
COLOR_YELLOW = (50, 35, 0)  # 6~8칸
COLOR_RED = (50, 0, 0)      # 9~10칸
COLOR_OFF = (0, 0, 0)

def set_leds(count):
    for i in range(LED_COUNT):
        if i < count:
            if i < 5:
                np[i] = COLOR_GREEN
            elif i < 8:
                np[i] = COLOR_YELLOW
            else:
                np[i] = COLOR_RED
        else:
            np[i] = COLOR_OFF
    np.write()

def blink_all_red(times=2, delay=0.5):
    for _ in range(times):
        for i in range(LED_COUNT):
            np[i] = COLOR_RED
        np.write()
        time.sleep(delay)
        
        for i in range(LED_COUNT):
            np[i] = COLOR_OFF
        np.write()
        time.sleep(delay)

# -------------------------------------------------------------
# STEP 1: 스마트 칼리브레이션 (초반 3초 버림 + 7초 요동 측정)
# -------------------------------------------------------------
print("=" * 50)
print("🤥 초민감 거짓말 탐지기 준비 중...")
print("1/2단계: 센서 히터 안정화 중 (3초)...")
print("=" * 50)

# 1. 초반 3초 안정화
start_time = time.time()
while time.time() - start_time < 3.0:
    set_leds(1)
    time.sleep(0.1)

print("\n2/2단계: 평상시 노이즈 및 기준값 분석 중 (7초)...")
print("센서에 손대지 말고 가만히 기다려주세요!")

# 2. 나머지 7초 동안 샘플링
samples = []
start_time = time.time()
while time.time() - start_time < 7.0:
    val = adc.read_u16()
    samples.append(val)
    
    current_sec = int((time.time() - start_time) * 2)
    set_leds((current_sec % LED_COUNT) + 1)
    time.sleep(0.1)

# 평상시 수치 분석
baseline = sum(samples) // len(samples)
min_noise = min(samples)
max_noise = max(samples)
noise_range = max_noise - min_noise

# 💡 [요청사항 반영]
# 평상시 요동 폭(noise_range)의 2배가 LED 10칸(최대치)이 되도록 설정
# 즉, 1칸당 민감도는 (noise_range * 2) / 10
target_max = noise_range * 2

# noise_range가 너무 작아서 0이 되는 것을 방지 (최소 target_max = 50 보장)
if target_max < 50:
    target_max = 50

sensitivity = target_max / 10.0 # 1칸당 필요한 변화량

set_leds(0)

print(f"\n✅ 분석 완료!")
print(f" - 평상시 평균(Baseline): {baseline}")
print(f" - 평상시 요동 폭: {noise_range}")
print(f" - LED 10칸(최대치) 목표 변화량: {target_max}")
print(f" - 1칸당 민감도: {sensitivity:.2f}")
print("\n이제 센서에 손가락을 대고 질문을 던져보세요! 😈\n")

# -------------------------------------------------------------
# STEP 2: 메인 탐지 루프
# -------------------------------------------------------------
try:
    while True:
        window_start = time.time()
        max_val = baseline
        
        while time.time() - window_start < 5.0:
            current_val = adc.read_u16()
            if current_val > max_val:
                max_val = current_val
            
            diff_temp = max_val - baseline
            if diff_temp < 0:
                diff_temp = 0
                
            temp_level = min(10, max(0, int(diff_temp // sensitivity)))
            set_leds(temp_level)
            
            time.sleep(0.05)
        
        delta = max_val - baseline
        if delta < 0:
            delta = 0
            
        level = min(10, max(0, int(delta // sensitivity)))
        set_leds(level)
        
        print(f"최대 값: {max_val} | 변화량(Delta): {delta} -> LED Level: {level}칸")
        
        if level > 8:
            print("🚨🚨 [거짓말 감지!] 당신은 거짓말을 하고 있습니다! 🚨🚨")
            blink_all_red(times=2, delay=0.5)
            set_leds(level)
        
        time.sleep(0.5)

except KeyboardInterrupt:
    set_leds(0)
    print("\n탐지기 종료")

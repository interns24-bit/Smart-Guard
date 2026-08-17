# Smart-Guard
import board, busio, time, array, audiobusio, math, pwmio, analogio, adafruit_ssd1306, neopixel
i2c = busio.I2C(board.GP5, board.GP4)
oled = adafruit_ssd1306.SSD1306_I2C(128, 64, i2c)
mic = audiobusio.PDMIn(board.GP3, board.GP2, sample_rate=16000, bit_depth=16)
samples = array.array('H', [0] * 1024)
pot = analogio.AnalogIn(board.GP28)
num_pixels = 5
pixel_pin = board.GP14
pixels = neopixel.NeoPixel(pixel_pin, num_pixels, brightness = 0.2)
buzzer = pwmio.PWMOut(board.GP21, variable_frequency=True)
def log10(x):
    return math.log(x) / math.log(10)

def normalized_rms(values):
    minbuf = sum(values) / len(values)
    samples_sum =sum(float(sample - minbuf) * (sample - minbuf)for sample in values)
    return math.sqrt(samples_sum / len(values))

def get_volume(sample_data):
    # Calculate the spread (peak-to-peak) of the sound wave
    return max(sample_data) - min(sample_data)

def trigger_alarm():
    oled.fill(0)
    oled.invert(True)
    oled.text("!! ALARM !!", 35, 10, 1)
    oled.text("DETECTED", 15, 35, 1, size=2)
    oled.show()
    
    for _ in range(5):
        buzzer.frequency = 4000
        buzzer.duty_cycle = 32768
        time.sleep(0.1)
        buzzer.frequency = 2000
        time.sleep(0.1)
        pixels.fill((255,0,0))
        time.sleep(0.5)
        pixels.fill((0,0,0))
        time.sleep(0.5)
        
    buzzer.duty_cycle = 0
    oled.invert(False)
    oled.show()
print("System Armed. Adjust Potentiometer for sensitivity.")

while True:
    # Read Potentiometer and convert to 0-100%
    # We invert it so turning right makes it MORE sensitive (lower threshold)
    pot_val = pot.value
    sensitivity_pct = int((pot_val / 65535) * 100)
    
    # Map sensitivity to a threshold (higher sensitivity = lower threshold)
    # Range 500 (very sensitive) to 10000 (very loud only)
    threshold = 10500 - (sensitivity_pct * 100)
    
    # Check Sound
    mic.record(samples, len(samples))
    magnitude = get_volume(samples)
    
    if magnitude > threshold:
        sound_level_dB = 20 * log10(magnitude)
        print(f"Sound Level (dB): {sound_level_dB:.2f}")
        trigger_alarm()
    else:
        print("Magnitude is too small to calculate dB.")
    time.sleep(0.1)
    
    # Update OLED Display
    oled.fill(0)
    oled.text("SECURITY SYSTEM", 20, 5, 1)
    oled.text("-" * 20, 5, 15, 1)
    oled.text(f"Sensitivity: {sensitivity_pct}%", 10, 30, 1)
    oled.text(f"Sound Lvl: {magnitude}", 10, 45, 1)
    
    # Draw a simple visual bar for sensitivity
    bar_width = int((sensitivity_pct / 100) * 100)
    oled.rect(10, 58, 100, 4, 1)
    oled.fill_rect(10, 58, bar_width, 4, 1)
    
    oled.show()
    time.sleep(0.05)  w

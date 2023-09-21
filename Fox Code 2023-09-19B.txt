"""
Code for W1MJ Fox 2.0
Updated 9/19/2023 by Eliot Mayer W1MJ
  - Added On Demand mode
  - Sync messages to start at start of minute (relative to Run button)
  - For battery voltage, now averaging multiple samples
  - After a period of inactivity, the message sequence starts from the beginning

Includes CircuitPython licenses:
   SPDX-FileCopyrightText: 2021 Kattni Rembor for Adafruit Industries
   SPDX-License-Identifier: MIT

"""

from analogio import AnalogIn
import audiomp3
import audiopwmio
import board
import digitalio
import math
import os
import time


###############################################
# SETTINGS
###############################################

t_message_interval_s = 60    # Time between start of messages (seconds)

# Default time of day [hour, min] prior to setting with pushbuttons
t_power_up_hr_min = [8, 0]

# ===== Scheduled Mode Settings =====
t_start_hr_min = [8, 0]  # Daily start time [hour, min]; use 5-minute intervals
t_stop_hr_min = [20, 0]  # Daily stop time [hour, min]; use 5-minute intervals

# ===== On Demand Mode Settings =====
t_on_demand_run_time = [0, 5]  # Run time [hours, minutes]; use 5-minute intervals
rx_detect_min_v = 0.25         # Minimum RX level for On Demand request in volts
rx_detect_min_t = 1.50         # Minimum RX time for On Demand request in seconds

# ===== Battery Settings =====
v_bat_min = 12.2             # Minimum battery voltage for transmitting
v_bat_correction = 0.985     # Battery voltage correction factor to match DMM


###############################################
# DEFINE FUNCTIONS
###############################################

def talk(msg, msg_folder):
    print(f"Number of Messages: {len(msg)}")
    ptt.value = True
    for i in range(len(msg)):
        msg_file = msg_folder + '/' + msg[i]
        if msg_folder == '/talk':
            msg_file = msg_file + '.mp3'
        print(f"Message {i} : {msg_file}")
        decoded_audio = audiomp3.MP3Decoder(open(msg_file, "rb"))
        pwm_audio.play(decoded_audio)
        while pwm_audio.playing:
            pass
    ptt.value = False


def announce_time(t):
    t_local = time.localtime(t)
    t_min = t_local.tm_min
    t_hour = t_local.tm_hour
    print(f"Announce Time (h:m): {t_hour}:{t_min}")
    msg = ['?'] * 3
    if t_hour >= 12:
        msg[2] = 'PM'
        t_hour = t_hour - 12
    else:
        msg[2] = 'AM'
    msg[0] = voice_hours[t_hour]
    msg[1] = voice_mins[t_min//5]
    talk(msg, '/talk')


def announce_battery_voltage(v):
    v = round(v, 1)
    vwhole = math.floor(v)
    vtenth = round(10*(v-math.floor(v)))
    msg = ['?'] * 5
    msg[0] = 'battery_voltage_is'
    msg[1] = voice_bat[vwhole]
    msg[2] = 'point'
    msg[3] = voice_bat[vtenth]
    msg[4] = 'volts'
    talk(msg, '/talk')


def measure_battery_voltage():
    n_samples = 10
    sum_of_samples = 0
    for i in range(n_samples):
        v_sample = bat_mon.value * v_bat_scaling
        # print(f"Battery Voltage Sample:  {v_sample}")
        sum_of_samples = sum_of_samples + v_sample
    v_avg = sum_of_samples / n_samples
    print(f"Battery Voltage (average of {n_samples} samples):  {v_avg}")
    return v_avg


def time_of_day_mins(t):
    if type(t) == int:
        t_local = time.localtime(t)
        return t_local.tm_hour * 60 + t_local.tm_min
    else:
        return t[0] * 60 + t[1]


def on_demand_startup():
    global t_start_mins, t_stop_mins
    t_now = time.time() + t_correction
    t_start = t_now
    t_stop = t_start + t_on_demand_run_time[0]*60*60 + t_on_demand_run_time[1]*60
    t_start_mins = time_of_day_mins(t_start)
    t_stop_mins = time_of_day_mins(t_stop)
    print("In on_demand_startup")
    print(f"t_start_mins, t_stop_mins = {t_start_mins}, {t_stop_mins}")
    msg = ['?'] * 1
    msg[0] = 'on_demand_intro'
    talk(msg, '/talk')
    announce_time(t_stop)


def await_on_demand_request():
    # Check for received audio (rx_detect)
    while True:
        if rx_detect.value * 3.3 / 65536 > rx_detect_min_v:  # Initial detection
          # Check multiple RX level samples to minimize false triggering
          n_samples = 5
          samples_over_min_v = 0
          for i in range(n_samples):
              t_sample = rx_detect_min_t / n_samples
              time.sleep(t_sample)
              if rx_detect.value * 3.3 / 65536 > rx_detect_min_v:
                  samples_over_min_v = samples_over_min_v + 1
          # If most of the samples are above the minimum level, request is good
          if samples_over_min_v / n_samples >= 0.75:
              # Wait for request to end before proceeding
              while rx_detect.value * 3.3 / 65536 > rx_detect_min_v:
                  pass
              time.sleep(1)
              break


###############################################
# INITIALIZATION
###############################################

# IO Port Assignments

# led = digitalio.DigitalInOut(board.LED)      # RPi Pico on-board LED; not used
# led.direction = digitalio.Direction.OUTPUT

ptt = digitalio.DigitalInOut(board.GP15)       # Pin 20
ptt.direction = digitalio.Direction.OUTPUT
ptt.value = False

pwm_audio = audiopwmio.PWMAudioOut(board.GP11) # Pin 15

pb_hour = digitalio.DigitalInOut(board.GP2)    # Pin 4
pb_hour.direction = digitalio.Direction.INPUT
pb_hour.pull = digitalio.Pull.UP

pb_min = digitalio.DigitalInOut(board.GP3)     # Pin 5
pb_min.direction = digitalio.Direction.INPUT
pb_min.pull = digitalio.Pull.UP

pb_run = digitalio.DigitalInOut(board.GP1)     # Pin 2
pb_run.direction = digitalio.Direction.INPUT
pb_run.pull = digitalio.Pull.UP

bat_mon = AnalogIn(board.A0)                   # 31 (GP.26 / ADC0)
rx_detect = AnalogIn(board.A1)                 # 32 (GP.27 / ADC1)

# Get file info from /message folder
message_list = os.listdir("/messages")
message_list = sorted(message_list)
num_messages = len(message_list)
print(f"There are {num_messages} files in the message folder.")

# Other initialization
v_bat_scaling = 0.00026285 * v_bat_correction
t_now = time.time() + t_power_up_hr_min[0]*60*60 + t_power_up_hr_min[1]*60
# Make sure we are using 5-minute intervals (unless we add mp3 files for every minute):
t_now = round(t_now/300) * 300
t_start_mins = time_of_day_mins(t_start_hr_min)
t_stop_mins = time_of_day_mins(t_stop_hr_min)


# Voice announcement files (all to have .mp3 appended by announcement functions)
voice_hours = ['12', '1', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11']
voice_mins = ['oclock', '05', '10', '15', '20',
              '25', '30', '35', '40', '45', '50', '55']
voice_bat = ['0', '1', '2', '3', '4', '5', '6', '7',
             '8', '9', '10', '11', '12', '13', '14', '15', '16']

# Initial message
v_bat = measure_battery_voltage()
announce_battery_voltage(v_bat)


###############################################
# SET TIME
###############################################

# Set time per Hour and Min buttons until Run button is pushed

announce_time(t_now)

while True:

    if not(pb_hour.value):     # Hour button pushed
        t_now = t_now + 60*60  # Add 1 hour
        announce_time(t_now)

    if not(pb_min.value):      # Min button pushed
        t_now = t_now + 60*5   # Add 5 minutes
        announce_time(t_now)

    if not(pb_run.value):      # Run button pushed
        time.sleep(1)
        if(pb_run.value):             # Short Run button push
            fox_mode = "Scheduled"
        else:                         # Long Run button push
            fox_mode = "On Demand"
        break

t_correction = t_now - time.time()
print("Finished setting time.")
print(f"Finished setting time.  t_correction = {t_correction}")
print(f"fox_mode = {fox_mode}")


###############################################
# MESSAGE LOOP
###############################################

# Loop through messages if (1) time is between t_start_hr_min & t_stop_hr_min,
# and (2) battery voltage is > v_bat_min

active_state = False   # This is used to starte with 1st message when going active.

if fox_mode == "On Demand":
    await_on_demand_request()
    on_demand_startup()

while True:
    t_now = time.time() + t_correction
    t_now_mins = time_of_day_mins(t_now)
    t_active = t_now_mins >= t_start_mins and t_now_mins <= t_stop_mins
    bat_ok = bat_mon.value * v_bat_scaling >= v_bat_min
    print("Message Main Loop")
    print(f"t_now_mins, t_start_mins, t_stop_mins = {t_now_mins}, {t_start_mins}, {t_stop_mins}")
    if t_active and bat_ok:
        if not(active_state):
            active_state = True
            msg_num = 0
        # Start message at to start of a minute (hh:mm:00)
        while time.localtime(t_now).tm_sec != 0:
            t_now = time.time() + t_correction
            pass
        t_msg_start = t_now
        if msg_num == num_messages:       # Announce battery voltage after last message
            v_bat = measure_battery_voltage()
            announce_battery_voltage(v_bat)
            msg_num = 0
        else:                            # Transmit regular message
            msg = ['?'] * 1
            msg[0] = message_list[msg_num]
            talk(msg, '/messages')
            msg_num = msg_num + 1
        # Wait until it is time for the next message
        while t_now < t_msg_start + t_message_interval_s:
            t_now = time.time() + t_correction
            pass
    elif fox_mode == "On Demand" and not(t_active):  # Wait for new On Demand request
        print("On Demand and not t_active; wait for next request")
        active_state = False
        await_on_demand_request()
        on_demand_startup()
    else:  # Scheduled mode, not active
        active_state = False 

# The End
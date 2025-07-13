import cv2
import math
import numpy as np
import mediapipe as mp
import pyautogui
import autopy
import tkinter as tk
from tkinter import messagebox

settings = {
    "enable_cursor": True,
    "enable_scroll": True,
    "cursor_sensitivity_x": [110, 620],
    "cursor_sensitivity_y": [20, 350]
}

class handDetector:
    def __init__(self, mode=False, maxHands=2, detectionCon=0.5, trackCon=0.5):
        self.mode = mode
        self.maxHands = maxHands
        self.detectionCon = detectionCon
        self.trackCon = trackCon
        self.mpHands = mp.solutions.hands
        self.hands = self.mpHands.Hands(
            static_image_mode=self.mode,
            max_num_hands=self.maxHands,
            min_detection_confidence=self.detectionCon,
            min_tracking_confidence=self.trackCon
        )
        self.mpDraw = mp.solutions.drawing_utils

    def findHands(self, img, draw=True):
        imgRGB = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
        self.results = self.hands.process(imgRGB)
        if self.results.multi_hand_landmarks:
            for handLms in self.results.multi_hand_landmarks:
                if draw:
                    self.mpDraw.draw_landmarks(img, handLms, self.mpHands.HAND_CONNECTIONS)
        return img

    def findPosition(self, img, handNo=0, draw=True, color=(255, 0, 255)):
        lmList = []
        if self.results.multi_hand_landmarks:
            myHand = self.results.multi_hand_landmarks[handNo]
            for id, lm in enumerate(myHand.landmark):
                h, w, c = img.shape
                cx, cy = int(lm.x * w), int(lm.y * h)
                lmList.append([id, cx, cy])
                if draw:
                    cv2.circle(img, (cx, cy), 5, color, cv2.FILLED)
        return lmList

def customization_page():
    global enable_cursor, enable_scroll

    def save_settings():
        settings["enable_cursor"] = enable_cursor.get()
        settings["enable_scroll"] = enable_scroll.get()
        messagebox.showinfo("Settings Saved", "Your settings have been saved!")
        customization_window.destroy()
        start_gesture_control()

    customization_window = tk.Tk()
    customization_window.title("Customization")
    customization_window.geometry("720x400")

    welcome_label = tk.Label(customization_window, text="Customize Your Settings", font=('Helvetica', 16))
    welcome_label.pack(pady=20)

    cursor_sensitivity_label_x = tk.Label(customization_window, text="Cursor Sensitivity X:")
    cursor_sensitivity_label_x.pack()
    cursor_sensitivity_x = tk.StringVar(value=f"{settings['cursor_sensitivity_x'][0]}-{settings['cursor_sensitivity_x'][1]}")
    cursor_sensitivity_entry_x = tk.Entry(customization_window, textvariable=cursor_sensitivity_x)
    cursor_sensitivity_entry_x.pack()

    cursor_sensitivity_label_y = tk.Label(customization_window, text="Cursor Sensitivity Y:")
    cursor_sensitivity_label_y.pack()
    cursor_sensitivity_y = tk.StringVar(value=f"{settings['cursor_sensitivity_y'][0]}-{settings['cursor_sensitivity_y'][1]}")
    cursor_sensitivity_entry_y = tk.Entry(customization_window, textvariable=cursor_sensitivity_y)
    cursor_sensitivity_entry_y.pack()

    enable_cursor = tk.BooleanVar(value=settings["enable_cursor"])
    enable_scroll = tk.BooleanVar(value=settings["enable_scroll"])

    tk.Checkbutton(customization_window, text="Enable Cursor", variable=enable_cursor).pack()
    tk.Checkbutton(customization_window, text="Enable Scroll", variable=enable_scroll).pack()
    save_button = tk.Button(customization_window, text="Save and Start", command=save_settings)
    save_button.pack(pady=10)

    customization_window.mainloop()

def start_gesture_control():
    wCam, hCam = 640, 480
    cap = cv2.VideoCapture(0)
    cap.set(3, wCam)
    cap.set(4, hCam)
    detector = handDetector()
    tipIds = [4, 8, 12, 16, 20]

    while True:
        success, img = cap.read()
        if not success:
            break

        img = cv2.flip(img, 1)
        img = detector.findHands(img)
        lmList = detector.findPosition(img, draw=False)
        fingers = []

        if len(lmList) != 0:
            fingers.append(1 if lmList[tipIds[0]][1] > lmList[tipIds[0] - 1][1] else 0)

            for id in range(1, 5):
                fingers.append(1 if lmList[tipIds[id]][2] < lmList[tipIds[id] - 2][2] else 0)

            if settings["enable_scroll"] and fingers == [0, 1, 0, 0, 0]:  
                pyautogui.scroll(300)
            elif settings["enable_scroll"] and fingers == [0, 1, 1, 0, 0]:  
                pyautogui.scroll(-300)

            if settings["enable_cursor"]:
                x1, y1 = lmList[8][1], lmList[8][2]
                w, h = autopy.screen.size()
                X = int(np.interp(x1, settings["cursor_sensitivity_x"], [0, w - 1]))
                Y = int(np.interp(y1, settings["cursor_sensitivity_y"], [0, h - 1]))

                x_thumb, y_thumb = lmList[4][1], lmList[4][2]
                x_middle, y_middle = lmList[12][1], lmList[12][2]
                length_left_click = math.hypot(x_thumb - x_middle, y_thumb - y_middle)
                if length_left_click < 30:
                    autopy.mouse.click(button=autopy.mouse.Button.LEFT)

                x_index, y_index = lmList[8][1], lmList[8][2]
                length_right_click = math.hypot(x_thumb - x_index, y_thumb - y_index)
                if length_right_click < 30:
                    autopy.mouse.click(button=autopy.mouse.Button.RIGHT)
                    
                autopy.mouse.move(X, Y)

        cv2.imshow("Gesture Control", img)
        if cv2.waitKey(1) & 0xFF == ord('x'):
            break

    cap.release()
    cv2.destroyAllWindows()

customization_page()

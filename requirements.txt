import numpy as np
import cv2
import tkinter as tk
from tkinter import ttk
from PIL import Image, ImageTk
import threading


class ColorDetectionApp:
    def __init__(self, window, window_title):
        self.window = window
        self.window.title(window_title)

        # Initialize variables
        self.is_detecting = False
        self.detection_thread = None
        self.stop_event = threading.Event()

        # Configure the main window
        self.window.configure(bg="#f0f0f0")
        self.window.resizable(width=True, height=True)

        # Create frames
        self.video_frame = ttk.Frame(window)
        self.video_frame.grid(row=0, column=0, padx=10, pady=10)

        self.control_frame = ttk.Frame(window)
        self.control_frame.grid(row=1, column=0, padx=10, pady=10)

        self.info_frame = ttk.Frame(window)
        self.info_frame.grid(row=0, column=1, rowspan=2, padx=10, pady=10, sticky="n")

        # Create canvas for video display
        self.canvas = tk.Canvas(self.video_frame, width=640, height=480)
        self.canvas.pack()

        # Draw detection box (centered rectangle)
        self.box_size = 100
        self.box_x1 = 320 - self.box_size // 2
        self.box_y1 = 240 - self.box_size // 2
        self.box_x2 = self.box_x1 + self.box_size
        self.box_y2 = self.box_y1 + self.box_size

        # Create control buttons
        self.btn_start = ttk.Button(self.control_frame, text="Start Detection", command=self.start_detection)
        self.btn_start.grid(row=0, column=0, padx=5, pady=5)

        self.btn_stop = ttk.Button(self.control_frame, text="Stop Detection", command=self.stop_detection)
        self.btn_stop.grid(row=0, column=1, padx=5, pady=5)
        self.btn_stop.config(state=tk.DISABLED)

        self.btn_exit = ttk.Button(self.control_frame, text="Exit", command=self.exit_app)
        self.btn_exit.grid(row=0, column=2, padx=5, pady=5)

        # Create info display
        self.color_info = ttk.Label(self.info_frame, text="No color detected", font=("Arial", 12))
        self.color_info.grid(row=0, column=0, padx=5, pady=5, sticky="w")

        self.rgb_info = ttk.Label(self.info_frame, text="RGB: (-, -, -)", font=("Arial", 12))
        self.rgb_info.grid(row=1, column=0, padx=5, pady=5, sticky="w")

        self.color_preview = tk.Canvas(self.info_frame, width=100, height=100, bg="#ffffff")
        self.color_preview.grid(row=2, column=0, padx=5, pady=5)

        # Initialize webcam
        self.cap = cv2.VideoCapture(0)
        if not self.cap.isOpened():
            self.show_error("Error: Could not open webcam")
            return

        # Start video display (without detection)
        self.update_frame()

        # Set the protocol for when the window is closed
        self.window.protocol("WM_DELETE_WINDOW", self.exit_app)

    def show_error(self, message):
        error_label = ttk.Label(self.window, text=message, foreground="red")
        error_label.grid(row=2, column=0, columnspan=2, padx=10, pady=10)

    def start_detection(self):
        if not self.is_detecting:
            self.is_detecting = True
            self.btn_start.config(state=tk.DISABLED)
            self.btn_stop.config(state=tk.NORMAL)
            self.stop_event.clear()
            self.detection_thread = threading.Thread(target=self.detection_loop)
            self.detection_thread.daemon = True
            self.detection_thread.start()

    def stop_detection(self):
        if self.is_detecting:
            self.stop_event.set()
            self.is_detecting = False
            self.btn_start.config(state=tk.NORMAL)
            self.btn_stop.config(state=tk.DISABLED)
            self.color_info.config(text="No color detected")
            self.rgb_info.config(text="RGB: (-, -, -)")
            self.color_preview.config(bg="#ffffff")

    def exit_app(self):
        self.stop_detection()
        if self.cap.isOpened():
            self.cap.release()
        self.window.destroy()

    def update_frame(self):
        ret, frame = self.cap.read()
        if ret:
            # Flip the frame horizontally for a more natural view
            frame = cv2.flip(frame, 1)

            # Draw detection box
            cv2.rectangle(frame, (self.box_x1, self.box_y1), (self.box_x2, self.box_y2), (0, 255, 0), 2)

            # Convert to RGB for display
            cv2image = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

            # Convert to PhotoImage
            img = Image.fromarray(cv2image)
            imgtk = ImageTk.PhotoImage(image=img)

            # Update canvas
            self.canvas.imgtk = imgtk
            self.canvas.create_image(0, 0, anchor=tk.NW, image=imgtk)

        # Repeat after 10ms if not stopped
        if not self.stop_event.is_set():
            self.window.after(10, self.update_frame)

    def detection_loop(self):
        while self.is_detecting and not self.stop_event.is_set():
            ret, frame = self.cap.read()
            if not ret:
                continue

            # Flip the frame horizontally
            frame = cv2.flip(frame, 1)

            # Extract the region inside the box
            roi = frame[self.box_y1:self.box_y2, self.box_x1:self.box_x2]

            # Calculate the average color in the ROI
            avg_color_per_row = np.average(roi, axis=0)
            avg_color = np.average(avg_color_per_row, axis=0)
            avg_color = avg_color.astype(int)

            # Get the B, G, R values (OpenCV uses BGR)
            b, g, r = avg_color

            # Convert to HSV for color naming
            hsv_roi = cv2.cvtColor(np.uint8([[avg_color]]), cv2.COLOR_BGR2HSV)[0][0]
            h, s, v = hsv_roi

            # Determine the color name based on HSV values
            color_name = self.determine_color(h, s, v)

            # Update info on the main thread
            self.window.after(0, self.update_color_info, color_name, r, g, b)

    def update_color_info(self, color_name, r, g, b):
        if self.is_detecting:
            self.color_info.config(text=f"Detected: {color_name}")
            self.rgb_info.config(text=f"RGB: ({r}, {g}, {b})")
            # Update color preview
            hex_color = f"#{r:02x}{g:02x}{b:02x}"
            self.color_preview.config(bg=hex_color)

    def determine_color(self, h, s, v):
        # Simple color determination based on HSV
        if s < 30 and v > 180:
            return "White"
        elif v < 50:
            return "Black"
        elif s < 50:
            return "Gray"

        # Brown is typically low in hue (reddish), moderate-high saturation, and low-moderate value
        if (0 <= h < 20) and (s > 50) and (50 <= v < 150):
            return "Brown"
        elif h < 10 or h > 170:
            return "Red"
        elif 10 <= h < 30:
            return "Orange"
        elif 30 <= h < 60:
            return "Yellow"
        elif 60 <= h < 90:
            return "Green"
        elif 90 <= h < 120:
            return "Cyan"
        elif 120 <= h < 150:
            return "Blue"
        elif 150 <= h < 170:
            return "Purple"
        else:
            return "Unknown"


if __name__ == "__main__":
    root = tk.Tk()
    app = ColorDetectionApp(root, "Color Detection App")
    root.mainloop()

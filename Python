import cv2 #OpenCV - image processing (Reads images, converts colors, applies filters)
import pytesseract #OCR engine (Reads text from images)
from ultralytics import YOLO #YOLO - Object detection (Detects license plate location)
from tkinter import * #GUI (Creates GUI windows and buttons)
from tkinter import filedialog, messagebox, simpledialog, ttk 
from PIL import Image, ImageTk #GUI (Converts images for display in GUI)
import numpy as np  #Array operations
import os #Checks if a file or folder exists on this operating system
import serial #UART communication (Sends data to FPGA via USB)
import time #Delays
import atexit #Cleanup on exit (Ensures UART closes properly when program exits)

#Initialization
MODEL_PATH = r"runs/detect/train/weights/best.pt" #YOLO
TESSERACT_PATH = r"C:\Program Files\Tesseract-OCR\tesseract.exe" #OCR
DEFAULT_BAUD = 9600
if os.path.exists(TESSERACT_PATH):
    pytesseract.pytesseract.tesseract_cmd = TESSERACT_PATH

#Global variables (Values won't change)
ser = None #serial port
fpga_connected = False #Connection status flag
last_detected_data = "" #Stores last extracted digits
model = None #YOLO model

#Automatically closes UART when program exits
def cleanup():
    global ser
    if ser: #If ser = 'None' than this statement will execute.
        try:
            ser.close()
        except: #If error detetcted while closing.
            pass
atexit.register(cleanup)

def clean_plate_text(text):
    text = text.upper().replace("IND", "") #Converts string in uppercase.
    return ''.join(ch for ch in text if ch.isalnum()).strip() #Only extracts numbers & alphabets & '.strip' will remove spaces.
#eg., IND MH 20 EE 7602, After clean_plate_text = MH20EE7602 
def extract_numbers_only(text):
    return ''.join(ch for ch in text if ch.isdigit())
#eg., MH20EE7602, After extract_numbers_only = 207602 
def extract_plate_data(plate_text):
    if len(plate_text) >= 10:
        extracted = plate_text[2:4] + plate_text[6:10]
    else:
        extracted = plate_text
    return extract_numbers_only(extracted)

def ocr_with_tesseract(cropped):
    try:
        gray = cv2.cvtColor(cropped, cv2.COLOR_BGR2GRAY) #Grayscale Conversion
        gray = cv2.bilateralFilter(gray, 11, 17, 17) #Smooths noise        
        config = "--psm 7 -c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
        #'--psm 7' treats image as single line text & 'whitelist' will reduce ambiguity eg., I & 1, Only mentioned alphabets & numbers will be recognized.
        #Segmentation: psm 7: single line, psm 8: Single word, psm 10: Single character.
        text = pytesseract.image_to_string(gray, config=config) #Tesseract analyzes pixel patterns
        #LSTM neural network - Predicts characters based on visual features learned from training data.
        return clean_plate_text(text)
    except:
        return ""

def detect_plate_yolo(img_path):
    global last_detected_data, model
    if model is None:
        return None, "", ""
    image = cv2.imread(img_path)
    results = model.predict(source=image, conf=0.4, verbose=False) #Returns bounding boxes with confidence scores
    #By decreasing confidence I can increase number of detections but accuracy will decrease.
    if len(results[0].boxes) == 0:
        return None, "", ""
    boxes = results[0].boxes.data.cpu().numpy()
    x1, y1, x2, y2 = map(int, boxes[np.argmax(boxes[:, 4])][:4]) #Select Best Detection according to confidence levels
    cropped = image[y1:y2, x1:x2]
    text = ocr_with_tesseract(cropped)
    data = extract_plate_data(text)
    last_detected_data = data
    return cropped, text, data

def toggle_fpga_connection():
    global ser, fpga_connected
    if fpga_connected:
        try:
            ser.close()
        except:
            pass
        ser = None
        fpga_connected = False
        conn_led.config(bg="red")
        connect_btn.config(text="Connect FPGA")
        return
    com_port = simpledialog.askstring("FPGA Connection", "Enter COM Port (e.g., COM5):", parent=root)
    if not com_port:
        return
    try:
        ser = serial.Serial(com_port.strip().upper(), DEFAULT_BAUD, timeout=1)
        time.sleep(0.1) #10ms delay
        fpga_connected = True
        conn_led.config(bg="green")
        connect_btn.config(text="Disconnect FPGA")
    except:
        messagebox.showwarning("Connection Failed", "Incorrect port.")
        fpga_connected = False
        conn_led.config(bg="red")
def send_to_fpga():
    global ser, fpga_connected, last_detected_data
    if not fpga_connected:
        messagebox.showwarning("Warning", "Please connect FPGA first.")
        return
    if not last_detected_data:
        messagebox.showwarning("Warning", "No data to send.")
        return
    try:
        #Send reset character 'R' first to clear the display
        ser.write(b'R')
        time.sleep(0.05) #50ms delay for FPGA to process reset
        
        #Send the actual data
        for ch in last_detected_data:
            ser.write(ch.encode())
            time.sleep(0.01)  #Wait 10ms between chars
        ser.flush() #Ensures all data sent
        messagebox.showinfo("Sent", f"Data sent: {last_detected_data}")
    except:
        messagebox.showwarning("Connection Lost", "Could not send data.")

#GUI FUNCTIONS
def load_model():
    global model
    try:
        model = YOLO(MODEL_PATH)
        print("✅ Model loaded successfully")
    except Exception as e:
        messagebox.showerror("Error", f"Model not loaded:\n{e}")

def upload_image():
    global last_detected_data
    path = filedialog.askopenfilename(filetypes=[("Images", "*.jpg *.jpeg *.png *.bmp")])
    if not path:
        return

    #Clear previous results
    main_img_label.config(image="", bg="#CFD8DC")
    plate_label.config(image="", bg="#CFD8DC")
    data_label.config(text="-", fg="#9E9E9E")
    send_btn.config(state=DISABLED)

    root.update() #Refresh GUI

    #Display uploaded image
    img = Image.open(path).resize((300, 180))
    img_tk = ImageTk.PhotoImage(img)
    main_img_label.config(image=img_tk)
    main_img_label.image = img_tk

    #Detect new plate
    cropped, text, data = detect_plate_yolo(path)

    if cropped is None:
        data_label.config(text="No Plate Found ❌", fg="red", font=("Segoe UI", 14, "bold"))
        send_btn.config(state=DISABLED)
        return

    #Show detected plate
    cropped_img = Image.fromarray(cv2.cvtColor(cropped, cv2.COLOR_BGR2RGB)).resize((200, 70))
    plate_img = ImageTk.PhotoImage(cropped_img)
    plate_label.config(image=plate_img)
    plate_label.image = plate_img

    # Update data label
    if data:
        data_label.config(text=data, fg="#00C853", font=("Consolas", 22, "bold"))
        send_btn.config(state=NORMAL)
    else:
        data_label.config(text="No Plate Found ❌", fg="red", font=("Segoe UI", 14, "bold"))
        send_btn.config(state=DISABLED)

root = Tk()
root.title("Number Plate Recognition")
root.geometry("700x750")
root.configure(bg="#F5F5F5")
root.resizable(True, True)

style = ttk.Style()
style.theme_use("clam")
style.configure("TButton", font=("Segoe UI", 10, "bold"), padding=5, background="#2196F3", foreground="white")
style.map("TButton", background=[("active", "#1976D2")])

container = Frame(root, bg="#F5F5F5")
container.pack(expand=True)

Label(container, text="Number Plate Recognition", 
      font=("Segoe UI", 18, "bold"), fg="#0D47A1", bg="#F5F5F5").pack(pady=20)

#FPGA section
fpga_frame = Frame(container, bg="#E0E0E0", bd=1, relief=GROOVE)
fpga_frame.pack(pady=10, ipadx=5, ipady=5)
Label(fpga_frame, text="FPGA:", font=("Segoe UI", 10, "bold"), bg="#E0E0E0").pack(side=LEFT, padx=10, pady=8)
conn_led = Label(fpga_frame, width=2, height=1, bg="red")
conn_led.pack(side=LEFT, padx=5)
connect_btn = ttk.Button(fpga_frame, text="Connect FPGA", command=toggle_fpga_connection)
connect_btn.pack(side=RIGHT, padx=10, pady=5)

#Upload button
upload_btn = ttk.Button(container, text="Upload Image", command=upload_image)
upload_btn.pack(pady=15)

#Image display
main_img_label = Label(container, bg="#CFD8DC", width=300, height=180, relief=SUNKEN)
main_img_label.pack(pady=10)

#Plate section
Label(container, text="Detected Plate:", font=("Segoe UI", 10, "bold"), bg="#F5F5F5").pack(pady=(10, 2))
plate_label = Label(container, bg="#CFD8DC", width=200, height=70, relief=SUNKEN)
plate_label.pack()

#Data display
Label(container, text="Extracted Data:", font=("Segoe UI", 10, "bold"), bg="#F5F5F5").pack(pady=(15, 3))
data_label = Label(container, text="-", font=("Consolas", 22, "bold"), bg="#F5F5F5", fg="#9E9E9E")
data_label.pack(pady=5)

#Send button
send_btn = ttk.Button(container, text="Send Data", command=send_to_fpga, state=DISABLED)
send_btn.pack(pady=10)

#Centered footer just below Send Data
footer = Label(container, text="", 
               font=("Segoe UI", 10, "bold"), fg="#616161", bg="#F5F5F5")
footer.pack(pady=(10, 15))

#Load YOLO model
root.after(100, load_model) #Wait 100ms
root.mainloop() #Start GUI 

"""
For Bilateral Filter: d = 11, sigmaColor = 17, sigmaSpace = 17

1)d = Diameter of the pixel neighborhood, Determines how many surrounding pixels influence the filtering.
Higher value → stronger smoothing.

2)sigmaColor = Controls how much color difference influences filtering
Low value = preserves details more
High value = more aggressive smoothing

3)sigmaSpace = Controls how far the smoothing spreads spatially.
Low = only nearby pixels affect each other
High = smoothing spreads further
"""

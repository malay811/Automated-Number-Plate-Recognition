# **Automated Number Plate Recognition (ANPR) System**

This hybrid system combines **software** with **FPGA hardware** to detect vehicle license plates.

---

## **Core Technologies**

### **Software Stack**

* **Language:** Python.
* **Detection:** YOLOv8 (custom-trained model).
  
**OCR:** Tesseract OCR (Recognition).



### **Hardware & Communication**

* **Hardware:** Verilog HDL implemented on a **Nexys A7 FPGA**.
* **Communication:** **9600 baud UART link** using a custom reset protocol ('R' command).

---

## **Key Performance Metrics**

| Metric | Value |
| --- | --- |
| **Detection Accuracy** | 92.3% |
| **Precision ($mAP@0.5$)** | 0.971 |
| **Training Dataset** | 8,121 instances |

---

## **Technical Workflow**

### **1. Software Detection & Processing**

* A custom-trained YOLOv8 model identifies and crops the plate region from uploaded images.
* Grayscale conversion and **bilateral filtering** are applied to smooth backgrounds and enhance character edges for the OCR engine.

### **2. Hardware Visualization**

*  Extracted numeric strings are sent via UART to the FPGA.


*  Digits are shown on a **1 kHz multiplexed** seven-segment display to ensure flicker-free output.
*  A **6-digit shift register** handles data storage, including automatic resets if an overflow (7th digit) occurs.

---

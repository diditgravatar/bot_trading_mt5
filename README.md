# bot_trading_mt5

Berikut adalah panduan langkah demi langkah untuk membuat aplikasi robot trading menggunakan **Python, MetaTrader 5 API, dan Tkinter**, beserta kebutuhan library dan pengaturannya.

---

## **1. Persiapan Awal**

### **A. Instalasi Python**
1. **Unduh dan Instal Python:**  
   Pastikan Python 3.8 atau yang lebih baru sudah terpasang. Unduh di [python.org](https://www.python.org/downloads/).
2. **Pastikan pip Terinstal:**  
   Cek dengan `pip --version`. Jika belum ada, tambahkan pip dengan menjalankan:
   ```bash
   python -m ensurepip --upgrade
   ```

### **B. Instal MetaTrader 5**
1. **Unduh dan Instal Aplikasi MetaTrader 5:**  
   [MetaTrader 5](https://www.metatrader5.com/) harus diinstal di komputer Anda.
2. **Pastikan Anda Memiliki Akun Trading:**  
   Anda membutuhkan kredensial akun (nomor akun, password, server).

---

## **2. Kebutuhan Library**

### **Library yang Dibutuhkan**
- **MetaTrader5:** Untuk menghubungkan Python ke MetaTrader 5.
- **pandas:** Untuk pengolahan data (perhitungan MA).
- **tkinter:** Untuk membuat antarmuka aplikasi.
- **threading:** (built-in) Untuk menjalankan proses robot secara paralel.
- **time:** (built-in) Untuk menambahkan jeda antara eksekusi trading.

### **Instalasi Library**
Jalankan perintah berikut di terminal:
```bash
pip install MetaTrader5 pandas
```

---

## **3. Struktur Folder**

Buat folder proyek seperti ini:
```
robot_trading/
│
├── main.py                 # Main file untuk menjalankan aplikasi
├── config/
│   └── settings.py         # Konfigurasi akun dan parameter
├── core/
│   ├── mt5_connector.py    # Koneksi ke MetaTrader 5
│   ├── data_handler.py     # Pengambilan dan analisis data
│   ├── order_executor.py   # Eksekusi order
├── ui/
│   └── app_ui.py           # Antarmuka Tkinter
├── logs/
│   └── app.log             # File log aktivitas
└── requirements.txt        # Daftar library
```

---

## **4. Langkah Implementasi**

### **A. Buat Folder dan File**
1. Buat folder `robot_trading/` dan sub-folder seperti struktur di atas.
2. Tambahkan file Python (`.py`) sesuai struktur.

---

### **B. Konfigurasi Akun**
Edit file `config/settings.py`:
```python
# Konfigurasi aplikasi
MT5_ACCOUNT = 12345678          # Nomor akun MetaTrader
MT5_PASSWORD = "your_password"  # Password akun
MT5_SERVER = "your_broker_server"  # Nama server broker
SYMBOL = "XAUUSD"               # Simbol yang ditradingkan
TIMEFRAME = 1                   # Timeframe (1 = M1)
VOLUME = 0.1                    # Volume lot
```

---

### **C. Koneksi ke MetaTrader 5**
File `core/mt5_connector.py` menginisialisasi koneksi ke MetaTrader 5:
```python
import MetaTrader5 as mt5

def initialize_mt5(account, password, server):
    if not mt5.initialize():
        return f"Failed to initialize MT5: {mt5.last_error()}"
    if not mt5.login(account, password=password, server=server):
        return f"Failed to login: {mt5.last_error()}"
    return f"Connected to MT5 account {account}"
```

---

### **D. Pengambilan Data dan Analisis**
File `core/data_handler.py` mengolah data harga:
```python
import pandas as pd
from MetaTrader5 import mt5

def fetch_data(symbol, timeframe, bars=200):
    rates = mt5.copy_rates_from_pos(symbol, timeframe, 0, bars)
    if rates is None:
        return None
    data = pd.DataFrame(rates)
    data['time'] = pd.to_datetime(data['time'], unit='s')
    return data

def calculate_ma(data, period):
    return data['close'].rolling(window=period).mean()

def generate_signals(data):
    data['MA50'] = calculate_ma(data, 50)
    data['MA200'] = calculate_ma(data, 200)
    data['Buy_Signal'] = (data['MA50'] > data['MA200']) & (data['MA50'].shift(1) <= data['MA200'].shift(1))
    data['Sell_Signal'] = (data['MA50'] < data['MA200']) & (data['MA50'].shift(1) >= data['MA200'].shift(1))
    return data
```

---

### **E. Eksekusi Order**
File `core/order_executor.py` mengatur eksekusi order:
```python
from MetaTrader5 import mt5

def place_order(symbol, action, volume):
    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": symbol,
        "volume": volume,
        "type": mt5.ORDER_TYPE_BUY if action == 'buy' else mt5.ORDER_TYPE_SELL,
        "price": mt5.symbol_info_tick(symbol).ask if action == 'buy' else mt5.symbol_info_tick(symbol).bid,
        "deviation": 10,
        "magic": 234000,
        "comment": "MA Crossover Bot",
    }
    result = mt5.order_send(request)
    if result.retcode != mt5.TRADE_RETCODE_DONE:
        return f"Order failed: {result.retcode}"
    return f"Order successful: {result}"
```

---

### **F. Antarmuka Tkinter**
File `ui/app_ui.py` menciptakan antarmuka dengan tombol:
```python
import tkinter as tk
from tkinter import scrolledtext
from threading import Thread, Event
from core.mt5_connector import initialize_mt5
from core.data_handler import fetch_data, generate_signals
from core.order_executor import place_order
from config.settings import MT5_ACCOUNT, MT5_PASSWORD, MT5_SERVER, SYMBOL, TIMEFRAME, VOLUME

def start_robot(log_area, stop_event):
    position = None

    while not stop_event.is_set():
        data = fetch_data(SYMBOL, TIMEFRAME, bars=500)
        if data is not None:
            data = generate_signals(data)
            last_row = data.iloc[-1]

            if last_row['Buy_Signal'] and position != 'buy':
                log_area.insert(tk.END, "Buy signal detected. Placing order...\n")
                log_area.insert(tk.END, place_order(SYMBOL, 'buy', VOLUME) + "\n")
                position = 'buy'
            elif last_row['Sell_Signal'] and position != 'sell':
                log_area.insert(tk.END, "Sell signal detected. Placing order...\n")
                log_area.insert(tk.END, place_order(SYMBOL, 'sell', VOLUME) + "\n")
                position = 'sell'

        log_area.see(tk.END)
        stop_event.wait(60)

def create_ui():
    root = tk.Tk()
    root.title("MA Crossover Trading Bot")
    root.geometry("600x400")

    log_area = scrolledtext.ScrolledText(root, wrap=tk.WORD, height=20)
    log_area.pack(padx=10, pady=10, fill=tk.BOTH, expand=True)

    stop_event = Event()

    start_button = tk.Button(
        root,
        text="Start Robot",
        command=lambda: Thread(target=start_robot, args=(log_area, stop_event)).start(),
    )
    start_button.pack(pady=5)

    stop_button = tk.Button(
        root,
        text="Stop Robot",
        command=lambda: stop_event.set(),
    )
    stop_button.pack(pady=5)

    root.mainloop()
```

---

### **G. Menjalankan Aplikasi**
Edit `main.py` untuk menjalankan aplikasi:
```python
from ui.app_ui import create_ui

if __name__ == "__main__":
    create_ui()
```

---

## **5. Jalankan Proyek**
1. Masuk ke folder proyek:
   ```bash
   cd robot_trading
   ```
2. Jalankan aplikasi:
   ```bash
   python main.py
   ```

---

### **6. Log Aktivitas**
Log akan dicatat di `logs/app.log` untuk memantau operasi.

Dengan langkah-langkah ini, robot trading Anda sudah siap berjalan. 

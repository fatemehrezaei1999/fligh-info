import os
import sqlite3
from tkinter import *
from tkinter import messagebox
from datetime import datetime, timedelta
from selenium import webdriver
from bs4 import BeautifulSoup
import re
import time

if os.path.exists('flights.db'):
    os.remove('flights.db')

def tomorrow_Nextweek():
    today = datetime.now()
    dates = [today + timedelta(days=i) for i in range(8)]
    return [date.strftime('%Y-%m-%d') for date in dates]

def store_flight_data(flight_info, date):
    conn = sqlite3.connect('flights.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS flights
                 (flight_no TEXT, price TEXT, flight_time TEXT, airline TEXT, date TEXT)''')
    c.execute("INSERT INTO flights VALUES (?, ?, ?, ?, ?)",
              (flight_info[0], flight_info[1], flight_info[2], flight_info[3], date))
    conn.commit()
    conn.close()

def web_request(date):
    conn = sqlite3.connect('flights.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS flights
                 (flight_no TEXT, price TEXT, flight_time TEXT, airline TEXT, date TEXT)''')
    conn.commit()
    conn.close()
    
    options = webdriver.FirefoxOptions()
    options.add_argument("--headless")
    browser = webdriver.Firefox(options=options)
    browser.get(f'https://safarmarket.com/flights/cIST-cTHR/{date}/0/allclasses/1adults/0children/0infants')
    time.sleep(5)
    
    soup = BeautifulSoup(browser.page_source, 'html.parser')
    flights = soup.find_all('div', class_='card oneWay')
    
    if not flights:
        conn = sqlite3.connect('flights.db')
        c = conn.cursor()
        c.execute("INSERT INTO flights VALUES (?, ?, ?, ?, ?)",
                 ('--', '--', '--', '--', date))
        conn.commit()
        conn.close()
        print(f"No flights found for {date}")
    else:
        for flight in flights:
            flight_info = [
                re.findall(r'<div class="leaveFlightNo".*?>(.*?)</div>', str(flight))[0],
                re.findall(r'<div><span>(.*?) </span>تومان</div>', str(flight))[0],
                re.findall(r'<div>([۰-۹]{1,2}:[۰-۹]{2})<div>IKA</div>', str(flight))[0],
                re.findall(r'[A-Za-z0-9\s]+', flight.find('div', class_='sellType').parent.text)[0]
            ]
            store_flight_data(flight_info, date)
    browser.quit()

def display_flight_data(dates_to_show=None):
    try:
        conn = sqlite3.connect('flights.db')
        c = conn.cursor()
        

        c.execute("SELECT name FROM sqlite_master WHERE type='table' AND name='flights'")
        if not c.fetchone():
            messagebox.showinfo("Info", "No flight data available")
            return
        
        if dates_to_show:
            query = "SELECT * FROM flights WHERE date IN ({}) ORDER BY date".format(",".join(["?"]*len(dates_to_show)))
            c.execute(query, dates_to_show)
        else:
            c.execute("SELECT * FROM flights ORDER BY date")
        
        records = c.fetchall()
        
        if not records:
            messagebox.showinfo("Info", "No flight data available for selected dates")
            return
        
    except sqlite3.Error as e:
        messagebox.showerror("Database Error", f"An error occurred: {str(e)}")
    finally:
        conn.close()
    root = Tk()
    root.title("Flight Information")
    root.geometry("1000x600")


    canvas = Canvas(root, borderwidth=0)
    scrollbar = Scrollbar(root, orient="vertical", command=canvas.yview)
    scrollable_frame = Frame(canvas)

    canvas.configure(yscrollcommand=scrollbar.set)
    canvas.pack(side="left", fill="both", expand=True)
    scrollbar.pack(side="right", fill="y")

    canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")


    def on_frame_configure(event):
        canvas.configure(scrollregion=canvas.bbox("all"))

    scrollable_frame.bind("<Configure>", on_frame_configure)

    columns = ["Flight No","Price", "Time","Airline", "Date"]
    for col, text in enumerate(columns):
        Label(scrollable_frame, text=text, bg="plum", font=('Arial', 10, 'bold')).grid(row=0, column=col, padx=10, pady=5)

    row = 1
    for record in records:
        for col, value in enumerate(record):
            Label(scrollable_frame, text=value, bg="ivory").grid(row=row, column=col, padx=10, pady=2)
        row += 1

    root.mainloop()

def display_flight_data():
    def on_submit():
        choice = var.get()
        if choice == "day":
            date = entry_day.get()
            if not date:
                messagebox.showerror("Error", "Please enter a date!")
                return
            dates_to_show = [date]
        elif choice == "period":
            start_date = entry_start.get()
            end_date = entry_end.get()
            if not start_date or not end_date:
                messagebox.showerror("Error", "Please enter both start and end dates!")
                return
            start = datetime.strptime(start_date, "%Y-%m-%d")
            end = datetime.strptime(end_date, "%Y-%m-%d")
            delta = end - start
            dates_to_show = [(start + timedelta(days=i)).strftime("%Y-%m-%d") for i in range(delta.days + 1)]
        else:
            messagebox.showerror("Error", "Invalid choice!")
            return
        
        choice_window.destroy()
        
        for date in dates_to_show:
            web_request(date)
        display_flight_data(dates_to_show)

    choice_window = Tk()
    choice_window.title("Select Option")
    choice_window.geometry("400x400")

    var = StringVar(value="day")

    Label(choice_window, text="Do you want to search for a specific day or a time period?").pack(pady=10)

    Radiobutton(choice_window, text="Specific Day", variable=var, value="day").pack()
    entry_day = Entry(choice_window)
    entry_day.pack()
    Label(choice_window, text="Format: YYYY-MM-DD (e.g., 2025-06-07)").pack()

    Radiobutton(choice_window, text="Time Period", variable=var, value="period").pack(pady=10)
    Label(choice_window, text="Start Date:").pack()
    entry_start = Entry(choice_window)
    entry_start.pack()
    Label(choice_window, text="End Date:").pack()
    entry_end = Entry(choice_window)
    entry_end.pack()
    Label(choice_window, text="Format: YYYY-MM-DD (e.g., 2025-06-07 to 2025-06-10)").pack()

    Button(choice_window, text="Submit", command=on_submit).pack(pady=20)

    choice_window.mainloop()

display_flight_data()

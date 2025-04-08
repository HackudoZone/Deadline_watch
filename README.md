import tkinter as tk
from tkinter import messagebox # Used for exit confirmation
import json
import os
from datetime import datetime
import random

# --- Configuration ---
DATA_FILE = "deadline_watch_data.json"
QUOTES_FILE = "quotes.txt" # Optional external file for custom quotes

# --- Default Data & Quotes ---
# (Quotes remain the same)
DEFAULT_QUOTES = [
    "🔥 Keep pushing forward!", "🚀 Your only limit is your mind.",
    "💡 Success is not final, failure is not fatal: It is the courage to continue that counts.",
    "🎯 Dreams don’t work unless you do.", "💪 Tough times don’t last, tough people do!",
    "🌟 Believe you can and you’re halfway there.", "🔥 The secret of getting ahead is getting started.",
    "💖 Do what you love, and you’ll never work a day in your life.",
    "🦸\u200d♂️ Act as if what you do makes a difference. It does!",
    "🚴\u200d♂️ Don’t watch the clock; do what it does. Keep going!"
]
SHORT_QUOTES = ["👍 Keep going!", "🔥 Never give up!", "💪 Stay strong!", "🚀 Aim high!"]
MEDIUM_QUOTES = ["⏳ Every minute counts!", "📈 Progress is progress, no matter how small."]
LONG_QUOTES = ["📜 Success is the sum of small efforts, repeated day in and day out.", "🏆 Hardships often prepare ordinary people for an extraordinary destiny."]
STORIES = ["🌟 A famous runner trained every day for years, failing countless times. One day, he won the marathon and became a legend.",
           "💡 Thomas Edison reportedly failed thousands of times before successfully inventing the light bulb. Persistence pays off!"]

# --- Global Variables ---
timer_flash_state = False # Tracks the flashing state for the timer
timer_update_id = None    # To store the id of the root.after call for timer updates
flash_update_id = None    # To store the id for the flashing update loop

# --- Functions ---

def load_quotes():
    """Loads quotes from QUOTES_FILE if it exists, otherwise uses DEFAULT_QUOTES."""
    if os.path.exists(QUOTES_FILE):
        try:
            with open(QUOTES_FILE, "r", encoding="utf-8") as file:
                quotes = [line.strip() for line in file.readlines() if line.strip()]
            return quotes if quotes else DEFAULT_QUOTES
        except Exception as e:
            print(f"Error loading quotes file '{QUOTES_FILE}': {e}")
            return DEFAULT_QUOTES
    return DEFAULT_QUOTES
# QUOTES = load_quotes() # Still seems unused for display

def load_data():
    """Loads data from DATA_FILE or returns defaults."""
    default_data = {
        "target_date": "2025-12-31 23:59:59",
        "goal": "My Awesome Goal!",
        "quote": random.choice(LONG_QUOTES),
        "story": random.choice(STORIES)
    }
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as file:
                data = json.load(file)
                for key, value in default_data.items():
                    data.setdefault(key, value)
                print("Data loaded successfully.")
                return data
        except json.JSONDecodeError:
            print(f"Warning: Error decoding JSON from {DATA_FILE}. Using default data.")
            return default_data
        except Exception as e:
            print(f"Warning: Error loading data file '{DATA_FILE}': {e}. Using default data.")
            return default_data
    else:
        print("Data file not found. Using default data.")
        return default_data

def save_data():
    """Saves the current target date and goal to DATA_FILE."""
    date_str = target_date_var.get()
    if not date_str:
        messagebox.showwarning("Save Error", "Cannot save: Target date is empty.")
        return
    try:
        datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S")
    except ValueError:
         messagebox.showwarning("Save Error", "Cannot save: Invalid date format.\nPlease use YYYY-MM-DD HH:MM:SS.")
         return

    data_to_save = { "target_date": date_str, "goal": goal_var.get() }
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as file:
            json.dump(data_to_save, file, indent=4)
        print("Data saved successfully.")
        # Simple status feedback
        status_label.config(text="Saved!", fg="lightgreen")
        root.after(2000, lambda: status_label.config(text="")) # Clear after 2s
    except Exception as e:
        print(f"Error saving data to '{DATA_FILE}': {e}")
        messagebox.showerror("Save Error", f"Could not save data to file:\n{e}")

def validate_date_format(current_value):
    """Provides visual feedback on date entry format validity."""
    try:
        datetime.strptime(current_value, "%Y-%m-%d %H:%M:%S")
        # Check if date_entry exists before configuring
        if 'date_entry' in globals() and date_entry.winfo_exists():
            date_entry.config(fg="#00FF00") # Green text
        return True
    except ValueError:
        if 'date_entry' in globals() and date_entry.winfo_exists():
            if current_value:
                 date_entry.config(fg="red") # Red text
            else:
                 date_entry.config(fg="white") # Default color
        return False

def flash_timer_label():
    """Toggles the color of the timer label for a flashing effect."""
    global timer_flash_state, flash_update_id
    if not root.winfo_exists(): return # Stop if window closed

    current_color = timer_label.cget("fg")
    new_color = "#1e1e1e" if current_color == "red" else "red" # Toggle between red and background
    timer_label.config(fg=new_color)
    timer_flash_state = not timer_flash_state
    # Schedule next flash toggle
    flash_update_id = root.after(500, flash_timer_label) # Flash every 500ms

def stop_flashing():
    """Stops the flashing effect and resets timer appearance."""
    global flash_update_id
    if flash_update_id:
        root.after_cancel(flash_update_id)
        flash_update_id = None
    # Reset appearance (color will be set by update_timer)
    timer_label.config(font=("Courier", 24, "bold"))


def update_timer():
    """Updates the countdown timer label every second."""
    global timer_update_id, flash_update_id
    if not root.winfo_exists(): return # Stop if window closed

    date_str = target_date_var.get()
    is_valid = validate_date_format(date_str)

    if not is_valid:
        stop_flashing() # Stop flashing if date becomes invalid
        if date_str:
            timer_label.config(text="❌ Invalid date format!", fg="yellow", font=("Courier", 24, "bold"))
        else:
            timer_label.config(text="⏳ Enter Deadline...", fg="gray", font=("Courier", 24, "bold"))
        timer_update_id = root.after(1000, update_timer) # Keep checking
        return

    try:
        target_time = datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S")
        current_time = datetime.now()
        remaining_time = target_time - current_time

        if remaining_time.total_seconds() > 0:
            # --- Countdown running ---
            stop_flashing() # Ensure flashing stops if date was changed back to future
            days, remainder_seconds = divmod(int(remaining_time.total_seconds()), 86400)
            hours, remainder_seconds = divmod(remainder_seconds, 3600)
            minutes, seconds = divmod(remainder_seconds, 60)
            timer_label.config(text=f"⏳ {days}d {hours}h {minutes}m {seconds}s left", fg="lightgreen", font=("Courier", 24, "bold"))
        else:
            # --- Time's up! ---
            timer_label.config(text="⏰ Time's up!", font=("Courier", 26, "bold")) # Keep text, font
            # Start flashing only if not already flashing
            if not flash_update_id:
                 flash_timer_label() # Start the flashing loop

    except Exception as e:
        print(f"Error updating timer: {e}")
        stop_flashing()
        timer_label.config(text="Error updating timer", fg="red")

    # Schedule the next timer update
    timer_update_id = root.after(1000, update_timer)


# Functions to update quotes/stories (remain the same, no saving)
def update_second_quote():
    if 'second_quote_label' in globals() and second_quote_label.winfo_exists():
        second_quote_label.config(text=f'🕒 {random.choice(SHORT_QUOTES)}')
    root.after(1000, update_second_quote)

def update_minute_quote():
    if 'minute_quote_label' in globals() and minute_quote_label.winfo_exists():
        minute_quote_label.config(text=f'🕰️ {random.choice(MEDIUM_QUOTES)}')
    root.after(60000, update_minute_quote)

def update_hour_quote():
    if 'quote_label' in globals() and quote_label.winfo_exists():
        quote_label.config(text=f'💡 "{random.choice(LONG_QUOTES)}"')
    root.after(3600000, update_hour_quote)

def update_story():
    if 'story_label' in globals() and story_label.winfo_exists():
        story_label.config(text=f'📖 {random.choice(STORIES)}')
    root.after(86400000, update_story)

def on_closing():
    """Handles window close event."""
    global timer_update_id, flash_update_id
    if messagebox.askokcancel("Quit", "Do you want to exit Deadline Watch?"):
        # Stop scheduled tasks to prevent errors after destroy
        if timer_update_id: root.after_cancel(timer_update_id)
        if flash_update_id: root.after_cancel(flash_update_id)
        # Add cancellation for other loops if necessary
        # Need to store their IDs like timer_update_id

        root.destroy()

# --- Main Application Logic ---
data = load_data()

# --- GUI Setup ---
root = tk.Tk()
root.title("Deadline Watch")
root.geometry("850x650")
root.configure(bg="#1e1e1e")
root.minsize(700, 550)

# --- Tkinter Variables ---
target_date_var = tk.StringVar(value=data["target_date"])
goal_var = tk.StringVar(value=data["goal"])
target_date_var.trace_add("write", lambda name, index, mode: validate_date_format(target_date_var.get()))

# --- Main Content Frame ---
content_frame = tk.Frame(root, bg="#1e1e1e")
content_frame.pack(pady=20, padx=20, fill=tk.BOTH, expand=True)

# --- Widgets ---
header_label = tk.Label(content_frame, text="📅 Deadline Watch Countdown ⏳", font=("Helvetica", 30, "bold"), fg="#FFA500", bg="#1e1e1e")
header_label.pack(pady=(0, 20))

timer_label = tk.Label(content_frame, text="⏳ Initializing...", font=("Courier", 24, "bold"), fg="#00FF00", bg="#1e1e1e")
timer_label.pack()

# --- Quotes & Stories Frame ---
quotes_frame = tk.Frame(content_frame, bg="#2a2a2a", relief=tk.SUNKEN, borderwidth=2)
quotes_frame.pack(pady=15, padx=10, fill=tk.X)
initial_quote = data.get("quote", f'💡 "{random.choice(LONG_QUOTES)}"')
initial_story = data.get("story", f'📖 {random.choice(STORIES)}')
quote_label = tk.Label(quotes_frame, text=initial_quote, font=("Helvetica", 16, "italic"), fg="#FFD700", bg="#2a2a2a", wraplength=750)
quote_label.pack(pady=10, padx=10)
story_label = tk.Label(quotes_frame, text=initial_story, font=("Helvetica", 14), fg="#00FFFF", bg="#2a2a2a", wraplength=750)
story_label.pack(pady=10, padx=10)

# Secondary quote labels
second_quote_label = tk.Label(content_frame, text="", font=("Helvetica", 12), fg="#FFFFFF", bg="#1e1e1e")
second_quote_label.pack(pady=5)
minute_quote_label = tk.Label(content_frame, text="", font=("Helvetica", 14), fg="#FFA500", bg="#1e1e1e")
minute_quote_label.pack(pady=5)

# --- Input Frame ---
input_frame = tk.Frame(content_frame, bg="#1e1e1e")
input_frame.pack(pady=15)
entry_label = tk.Label(input_frame, text="Deadline (YYYY-MM-DD HH:MM:SS):", font=("Helvetica", 14), fg="#FFFFFF", bg="#1e1e1e")
entry_label.grid(row=0, column=0, padx=5, pady=5, sticky="e")
date_entry = tk.Entry(input_frame, textvariable=target_date_var, font=("Helvetica", 14), width=25, bg="#333333", fg="white", insertbackground="white")
date_entry.grid(row=0, column=1, padx=5, pady=5)
validate_date_format(target_date_var.get()) # Initial check
goal_label = tk.Label(input_frame, text="Your Goal:", font=("Helvetica", 14), fg="#FFFFFF", bg="#1e1e1e")
goal_label.grid(row=1, column=0, padx=5, pady=5, sticky="e")
goal_entry = tk.Entry(input_frame, textvariable=goal_var, font=("Helvetica", 14), width=40, bg="#333333", fg="white", insertbackground="white")
goal_entry.grid(row=1, column=1, padx=5, pady=5)

# --- Buttons Frame ---
buttons_frame = tk.Frame(content_frame, bg="#1e1e1e")
buttons_frame.pack(pady=10)
save_button = tk.Button(buttons_frame, text="💾 Save Goal & Deadline", command=save_data, font=("Helvetica", 14, "bold"), bg="#4CAF50", fg="white", relief=tk.RAISED, borderwidth=3, padx=10)
save_button.grid(row=0, column=0, padx=10)
quit_button = tk.Button(buttons_frame, text="❌ Quit", command=on_closing, font=("Helvetica", 14, "bold"), bg="#f44336", fg="white", relief=tk.RAISED, borderwidth=3, padx=10)
quit_button.grid(row=0, column=1, padx=10)

# Status Label for feedback
status_label = tk.Label(content_frame, text="", font=("Helvetica", 12), fg="lightgreen", bg="#1e1e1e")
status_label.pack(pady=(5, 0))


# --- Start Update Loops & Main Loop ---
root.protocol("WM_DELETE_WINDOW", on_closing)
update_timer() # Start the main timer loop
update_second_quote()
update_minute_quote()
update_hour_quote()
update_story()
root.mainloop()

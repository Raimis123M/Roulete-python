import tkinter as tk
import math, random

NUMBERS = [0,32,15,19,4,21,2,25,17,34,6,27,13,36,11,30,8,23,10,5,24,16,33,1,20,14,31,9,22,18,29,7,28,12,35,3,26]
RED = {1,3,5,7,9,12,14,16,18,19,21,23,25,27,30,32,34,36}
SLICE = 360 / len(NUMBERS)
CX, CY, R = 200, 200, 170

def color_of(n):
    if n == 0: return "#008c3c"
    return "#c81e1e" if n in RED else "#141414"

root = tk.Tk()
root.title("Roulette")
canvas = tk.Canvas(root, width=940, height=400, bg="#0a5028", highlightthickness=0)
canvas.pack()

# ---------- wheel ----------
arcs = [canvas.create_arc(CX-R, CY-R, CX+R, CY+R, start=i*SLICE, extent=SLICE,
                          fill=color_of(n), outline="#d4af37")
        for i, n in enumerate(NUMBERS)]
canvas.create_oval(CX-35, CY-35, CX+35, CY+35, fill="#222", outline="#d4af37", width=2)
ball = canvas.create_oval(0, 0, 0, 0, fill="white")

# ---------- betting table ----------
cells = {}   # key -> (x1, y1, x2, y2)
TX, TY, CW, CH = 440, 60, 40, 50
cells[0] = (TX - CW, TY, TX, TY + 3 * CH)
for col in range(12):
    for row in range(3):
        n = col * 3 + (3 - row)          # top row = 3,6,9... like a real table
        x1, y1 = TX + col * CW, TY + row * CH
        cells[n] = (x1, y1, x1 + CW, y1 + CH)
for i, name in enumerate(["1-18", "EVEN", "RED", "BLACK", "ODD", "19-36"]):
    x1, y1 = TX + i * 80, TY + 3 * CH + 10
    cells[name] = (x1, y1, x1 + 80, y1 + CH)

for key, (x1, y1, x2, y2) in cells.items():
    if isinstance(key, int):
        fill = color_of(key)
    else:
        fill = {"RED": "#c81e1e", "BLACK": "#141414"}.get(key, "#0a5028")
    canvas.create_rectangle(x1, y1, x2, y2, fill=fill, outline="white")
    canvas.create_text((x1+x2)/2, (y1+y2)/2, text=str(key), fill="white",
                       font=("Arial", 11, "bold"))

balance_text = canvas.create_text(440, 15, anchor="nw", fill="gold", font=("Arial", 18, "bold"))
status_text = canvas.create_text(440, 300, anchor="nw", fill="white", font=("Arial", 18),
                                 text="Pick a chip, click the table, then SPIN")

# ---------- game state ----------
state = {"wheel": 0.0, "ball": 0.0, "speed": 0.0, "spinning": False,
         "balance": 500, "chip": 10}
bets = {}    # key -> amount

def refresh_balance():
    canvas.itemconfigure(balance_text, text=f"Balance: ${state['balance']}")

def redraw_chips():
    canvas.delete("chip")
    for key, amt in bets.items():
        x1, y1, x2, y2 = cells[key]
        cx, cy = (x1 + x2) / 2, (y1 + y2) / 2
        canvas.create_oval(cx-14, cy-14, cx+14, cy+14, fill="gold", outline="black", tags="chip")
        canvas.create_text(cx, cy, text=str(amt), font=("Arial", 9, "bold"), tags="chip")

def on_click(event):
    if state["spinning"]:
        return
    for key, (x1, y1, x2, y2) in cells.items():
        if x1 <= event.x < x2 and y1 <= event.y < y2:
            if state["balance"] >= state["chip"]:
                state["balance"] -= state["chip"]
                bets[key] = bets.get(key, 0) + state["chip"]
                redraw_chips()
                refresh_balance()
            return

def clear_bets():
    if state["spinning"]:
        return
    state["balance"] += sum(bets.values())
    bets.clear()
    redraw_chips()
    refresh_balance()

# ---------- spinning ----------
def place_ball():
    r = R - 15
    x = CX + r * math.cos(math.radians(state["ball"]))
    y = CY - r * math.sin(math.radians(state["ball"]))
    canvas.coords(ball, x-8, y-8, x+8, y+8)

def wins(key, n):
    if isinstance(key, int): return key == n
    if n == 0: return False
    return {"RED": n in RED, "BLACK": n not in RED, "EVEN": n % 2 == 0,
            "ODD": n % 2 == 1, "1-18": n <= 18, "19-36": n >= 19}[key]

def finish():
    rel = (state["ball"] - state["wheel"]) % 360
    n = NUMBERS[int(rel / SLICE) % len(NUMBERS)]
    won = sum(amt * (36 if isinstance(k, int) else 2)
              for k, amt in bets.items() if wins(k, n))
    state["balance"] += won
    canvas.itemconfigure(status_text, text=f"Result: {n}   |   You won: ${won}")
    bets.clear()
    redraw_chips()
    refresh_balance()

def update():
    s = state
    s["wheel"] += s["speed"] * 0.3
    s["ball"] -= s["speed"]
    s["speed"] *= 0.99
    for i, arc in enumerate(arcs):
        canvas.itemconfigure(arc, start=s["wheel"] + i * SLICE)
    place_ball()
    if s["speed"] < 0.05:
        s["spinning"] = False
        finish()
    else:
        root.after(16, update)

def spin(event=None):
    if state["spinning"] or not bets:
        return
    state["spinning"] = True
    state["speed"] = random.uniform(12, 18)
    canvas.itemconfigure(status_text, text="Spinning...")
    update()

# ---------- controls ----------
def set_chip(v):
    state["chip"] = v
    canvas.itemconfigure(status_text, text=f"Chip set to ${v}")

bar = tk.Frame(root)
bar.pack(pady=6)
for v in (5, 10, 25, 100):
    tk.Button(bar, text=f"${v}", width=6, command=lambda v=v: set_chip(v)).pack(side="left", padx=3)
tk.Button(bar, text="CLEAR BETS", command=clear_bets).pack(side="left", padx=12)
tk.Button(bar, text="SPIN", width=10, command=spin).pack(side="left", padx=3)

canvas.bind("<Button-1>", on_click)
root.bind("<space>", spin)
refresh_balance()
place_ball()
root.mainloop()

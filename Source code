# -*- coding: utf-8 -*-
# MexicoHurricaneTree — English version

import math
import re
import threading
import time
import tkinter as tk
import random
import requests

NHC_URL = "https://www.nhc.noaa.gov/CurrentStorms.json"
OPEN_METEO_URL = "https://api.open-meteo.com/v1/forecast"

MEXICO_LAT_MIN = 5
MEXICO_LAT_MAX = 35
MEXICO_LON_MIN = -120
MEXICO_LON_MAX = -75

REFRESH_SECONDS = 10 * 60
FRAME_DELAY_MS = 16


class MexicoHurricaneTree:
    def __init__(self, root):
        self.root = root
        self.root.title("Storm Watch — Mexico")
        self.root.geometry("1280x800")
        self.root.minsize(980, 650)
        self.root.configure(bg="#07111d")

        self.canvas = tk.Canvas(
            root,
            bg="#07111d",
            highlightthickness=0
        )
        self.canvas.pack(fill="both", expand=True)

        self.data = {
            "status": "loading",
            "name": "",
            "wind": 0.0,
            "gust": 0.0,
            "direction": None,
            "lat": None,
            "lon": None,
            "note": "Connecting to the hurricane monitoring center..."
        }

        self.loading = False
        self.last_refresh = 0
        self.phase = 0.0
        self.wind_phase = 0.0
        self.last_frame_time = time.perf_counter()

        self.lightning_active = False
        self.last_lightning_time = 0
        self.lightning_timer = 0
        self.lightning_duration = 0.16

        self.debris = []
        self.max_debris = 18
        self.debris_spawn_timer = 0.0

        self.rain = []
        self.max_rain = 120
        self.rain_initialized = False

        self.scene_box = None
        self.refresh_box = (0, 0, 0, 0)

        self.root.bind("<Configure>", self.on_resize)
        self.root.bind("<r>", lambda _event: self.refresh())
        self.root.bind("<R>", lambda _event: self.refresh())
        self.canvas.bind("<Button-1>", self.on_click)

        self.refresh()
        self.animate()

    @staticmethod
    def get_number(value):
        if value is None:
            return None
        match = re.search(r"-?\d+(?:\.\d+)?", str(value))
        if not match:
            return None
        try:
            return float(match.group(0))
        except ValueError:
            return None

    @staticmethod
    def get_coordinate(value):
        if value is None:
            return None
        text = str(value).strip().upper()
        number = MexicoHurricaneTree.get_number(text)
        if number is None:
            return None
        if "S" in text or "W" in text:
            number = -abs(number)
        elif "N" in text or "E" in text:
            number = abs(number)
        return number

    @staticmethod
    def in_mexico_region(lat, lon):
        return (
            MEXICO_LAT_MIN <= lat <= MEXICO_LAT_MAX
            and MEXICO_LON_MIN <= lon <= MEXICO_LON_MAX
        )

    def get_nhc_storms(self):
        response = requests.get(
            NHC_URL, timeout=20,
            headers={"User-Agent": "MexicoHurricaneTree/2.0"}
        )
        response.raise_for_status()
        payload = response.json()

        if isinstance(payload, dict):
            storms = payload.get("activeStorms", [])
        elif isinstance(payload, list):
            storms = payload
        else:
            storms = []

        if isinstance(storms, dict):
            storms = list(storms.values())

        results = []
        for storm in storms:
            if not isinstance(storm, dict):
                continue
            classification = str(
                storm.get("classification", storm.get("class", ""))
            ).upper()
            intensity_text = storm.get(
                "intensity", storm.get("wind", storm.get("maxWind", ""))
            )
            wind_knots = self.get_number(intensity_text)

            is_hurricane = (
                "HU" in classification
                or "HURRICANE" in classification
                or (wind_knots is not None and wind_knots >= 64)
            )
            if not is_hurricane:
                continue

            lat = self.get_coordinate(storm.get("latitude", storm.get("lat")))
            lon = self.get_coordinate(storm.get("longitude", storm.get("lon")))
            if lat is None or lon is None:
                continue
            if not self.in_mexico_region(lat, lon):
                continue

            wind_kmh = (wind_knots or 0) * 1.852
            results.append({
                "name": str(storm.get("name", "Hurricane")),
                "lat": lat, "lon": lon,
                "wind_kmh": wind_kmh,
                "classification": classification
            })
        return results

    @staticmethod
    def get_weather(lat, lon):
        params = {
            "latitude": lat, "longitude": lon,
            "current": "wind_speed_10m,wind_gusts_10m,wind_direction_10m",
            "wind_speed_unit": "kmh", "timezone": "UTC"
        }
        response = requests.get(
            OPEN_METEO_URL, params=params, timeout=20,
            headers={"User-Agent": "MexicoHurricaneTree/2.0"}
        )
        response.raise_for_status()
        current = response.json().get("current", {})
        wind = current.get("wind_speed_10m")
        gust = current.get("wind_gusts_10m")
        direction = current.get("wind_direction_10m")
        return (
            float(wind) if wind is not None else None,
            float(gust) if gust is not None else None,
            float(direction) if direction is not None else None
        )

    def load_data_worker(self):
        try:
            storms = self.get_nhc_storms()
            if not storms:
                result = {
                    "status": "calm", "name": "", "wind": 0.0, "gust": 0.0,
                    "direction": None, "lat": None, "lon": None,
                    "note": "No active hurricane detected near Mexico."
                }
            else:
                storm = max(storms, key=lambda item: item["wind_kmh"])
                wind = storm["wind_kmh"]
                gust = wind * 1.30
                direction = None
                note = "Wind speed from NHC. Gusts are estimated."
                try:
                    model_wind, model_gust, direction = self.get_weather(
                        storm["lat"], storm["lon"]
                    )
                    if model_wind is not None:
                        wind = max(wind, model_wind)
                    if model_gust is not None:
                        gust = max(model_gust, wind)
                    note = "Data cross-checked: NHC + Open-Meteo."
                except Exception:
                    pass
                result = {
                    "status": "hurricane", "name": storm["name"],
                    "wind": wind, "gust": max(gust, wind),
                    "direction": direction,
                    "lat": storm["lat"], "lon": storm["lon"],
                    "note": note
                }
        except Exception as error:
            result = {
                "status": "error", "name": "", "wind": 0.0, "gust": 0.0,
                "direction": None, "lat": None, "lon": None,
                "note": f"Unable to retrieve data: {error}"
            }

        self.data = result
        self.loading = False
        try:
            self.root.after(16, self.draw_static)
        except tk.TclError:
            pass

    def refresh(self):
        if self.loading:
            return
        self.loading = True
        self.last_refresh = time.time()
        threading.Thread(target=self.load_data_worker, daemon=True).start()
        self.draw_static()

    def wind_strength(self):
        wind = max(
            self.data.get("wind", 0) or 0,
            self.data.get("gust", 0) or 0
        )
        return min(1.0, wind / 190.0)

    def wind_direction_vector(self):
        direction = self.data.get("direction")
        if direction is None:
            return 1.0, 0.0
        downwind = math.radians((direction + 180) % 360)
        return math.sin(downwind), -math.cos(downwind)

    def trigger_lightning(self):
        if self.data["status"] != "hurricane":
            self.lightning_active = False
            return
        now = time.time()
        interval = random.uniform(4.0, 13.0)
        if now - self.last_lightning_time > interval:
            self.last_lightning_time = now
            self.lightning_active = True
            self.lightning_timer = now

    def draw_lightning(self, c, x1, y1, x2, y2):
        segments = random.randint(5, 8)
        points = [(x1, y1)]
        width = max(20, abs(x2 - x1))
        height = max(30, abs(y2 - y1))
        for i in range(1, segments):
            ratio = i / segments
            px = x1 + ratio * (x2 - x1) + random.uniform(-width * 0.18, width * 0.18)
            py = y1 + ratio * (y2 - y1) + random.uniform(-height * 0.08, height * 0.08)
            points.append((px, py))
        points.append((x2, y2))
        for i in range(len(points) - 1):
            xs, ys = points[i]
            xe, ye = points[i + 1]
            c.create_line(xs, ys, xe, ye, fill="#dffaff", width=8,
                          capstyle=tk.ROUND, tags="lightning")
            c.create_line(xs, ys, xe, ye, fill="#ffffff", width=3,
                          capstyle=tk.ROUND, tags="lightning")

    def init_rain(self):
        if self.rain_initialized or not self.scene_box:
            return
        x1, y1, x2, y2 = self.scene_box
        self.rain.clear()
        for _ in range(self.max_rain):
            self.rain.append([
                random.uniform(x1, x2),
                random.uniform(y1, y2),
                random.uniform(350, 700),
                random.uniform(8, 18),
                random.uniform(0.2, 1.0)
            ])
        self.rain_initialized = True

    def update_rain(self, delta):
        if not self.scene_box:
            return
        if self.data["status"] != "hurricane":
            return
        self.init_rain()
        x1, y1, x2, y2 = self.scene_box
        wind = self.data.get("wind", 0) or 0
        direction_x, _ = self.wind_direction_vector()
        for r in self.rain:
            r[0] += (direction_x * (wind * 0.25) + wind * 0.02) * delta
            r[1] += r[2] * delta
            if r[1] > y2 + 30:
                r[0] = random.uniform(x1, x2)
                r[1] = y1 - random.uniform(10, 150)
            if r[0] < x1 - 100 or r[0] > x2 + 100:
                r[0] = random.uniform(x1, x2)

    def draw_rain(self, c):
        if self.data["status"] != "hurricane":
            return
        self.init_rain()
        wind = self.data.get("wind", 0) or 0
        direction_x, _ = self.wind_direction_vector()
        for x, y, speed, length, opacity in self.rain:
            slant = direction_x * (12 + wind * 0.06)
            c.create_line(x, y, x + slant, y + length,
                         fill="#8db5c8", width=1, tags="rain")

    def spawn_debris(self):
        if self.data["status"] != "hurricane":
            return
        if len(self.debris) >= self.max_debris:
            return
        if not self.scene_box:
            return
        x1, y1, x2, y2 = self.scene_box
        w = x2 - x1
        h = y2 - y1
        direction_x, direction_y = self.wind_direction_vector()
        x = x1 + random.uniform(0, w)
        y = y1 + random.uniform(-h * 0.15, h * 0.55)
        wind = self.data.get("wind", 0) or 0
        gust = self.data.get("gust", 0) or wind
        speed = 20 + wind * random.uniform(0.45, 0.9)
        vx = direction_x * speed + random.uniform(-30, 30)
        vy = direction_y * speed + random.uniform(5, 35)
        size = random.uniform(3, 9)
        color = random.choice(["#6f5a45", "#806a51", "#4d4033", "#93795a", "#5c4a39"])
        life = random.uniform(1.0, 3.0)
        self.debris.append([
            x, y, vx, vy, size, color, life,
            random.uniform(0, math.pi * 2),
            random.uniform(-4, 4)
        ])

    def update_debris(self, delta):
        if self.data["status"] != "hurricane":
            self.debris.clear()
            return
        if not self.scene_box:
            return
        x1, y1, x2, y2 = self.scene_box
        to_remove = []
        for d in self.debris:
            d[0] += d[2] * delta
            d[1] += d[3] * delta
            d[7] += d[8] * delta
            d[6] -= delta
            if (d[6] <= 0 or d[0] < x1 - 100 or d[0] > x2 + 100 or d[1] > y2 + 100):
                to_remove.append(d)
        for d in to_remove:
            if d in self.debris:
                self.debris.remove(d)

    def draw_debris(self, c):
        for d in self.debris:
            x, y, _, _, size, color, _, rotation, _ = d
            points = []
            for i in range(4):
                angle = rotation + math.pi / 4 + i * math.pi / 2
                radius = size * (1.0 if i % 2 == 0 else 0.65)
                points.extend([
                    x + math.cos(angle) * radius,
                    y + math.sin(angle) * radius
                ])
            c.create_polygon(points, fill=color, outline="", tags="debris")

    def rounded_rect(self, c, x1, y1, x2, y2, radius=14, fill="#101c2a", outline=""):
        c.create_rectangle(x1 + radius, y1, x2 - radius, y2, fill=fill, outline="")
        c.create_rectangle(x1, y1 + radius, x2, y2 - radius, fill=fill, outline="")
        c.create_arc(x1, y1, x1 + radius * 2, y1 + radius * 2,
                     start=90, extent=90, fill=fill, outline="")
        c.create_arc(x2 - radius * 2, y1, x2, y1 + radius * 2,
                     start=0, extent=90, fill=fill, outline="")
        c.create_arc(x1, y2 - radius * 2, x1 + radius * 2, y2,
                     start=180, extent=90, fill=fill, outline="")
        c.create_arc(x2 - radius * 2, y2 - radius * 2, x2, y2,
                     start=270, extent=90, fill=fill, outline="")
        if outline:
            c.create_rectangle(x1 + radius, y1, x2 - radius, y2,
                               outline=outline, width=1, fill="")

    def status_color(self):
        if self.loading:
            return "#f3c969"
        if self.data["status"] == "hurricane":
            return "#ff6b61"
        if self.data["status"] == "calm":
            return "#45d19b"
        return "#f3c969"

    def draw_wind_gauge(self, cx, cy, radius):
        c = self.canvas
        wind = self.data.get("wind", 0) or 0
        gust = self.data.get("gust", 0) or 0
        value = min(1.0, max(wind, gust) / 190.0)

        c.create_oval(cx - radius, cy - radius, cx + radius, cy + radius,
                      fill="#0b1724", outline="#25394b", width=2, tags="static_ui")

        for i in range(31):
            angle = math.radians(-140 + i * (280 / 30))
            inner = radius - 12
            outer = radius - (5 if i % 5 else 1)
            x1 = cx + math.cos(angle) * inner
            y1 = cy + math.sin(angle) * inner
            x2 = cx + math.cos(angle) * outer
            y2 = cy + math.sin(angle) * outer
            c.create_line(x1, y1, x2, y2,
                          fill="#496274" if i % 5 else "#8298a8",
                          width=1 if i % 5 else 2, tags="static_ui")

        c.create_arc(cx - radius + 7, cy - radius + 7,
                      cx + radius - 7, cy + radius - 7,
                      start=140, extent=-280 * value, style=tk.ARC,
                      outline="#ff7569" if value > 0.65 else "#55d3a0",
                      width=8, tags="static_ui")

        c.create_text(cx, cy - 8, text=f"{wind:.0f}",
                      fill="#f3f7fa", font=("Segoe UI", 24, "bold"), tags="static_ui")
        c.create_text(cx, cy + 19, text="KM/H",
                      fill="#839aa9", font=("Segoe UI", 8, "bold"), tags="static_ui")
        c.create_text(cx, cy + radius + 18, text=f"GUSTS  {gust:.0f} km/h",
                      fill="#9fb1bd", font=("Segoe UI", 8, "bold"), tags="static_ui")

    def draw_static(self):
        if not self.root.winfo_exists():
            return
        c = self.canvas
        c.delete("all")

        width = max(980, c.winfo_width())
        height = max(650, c.winfo_height())

        c.create_rectangle(0, 0, width, height, fill="#07111d", outline="")

        c.create_rectangle(0, 0, width, 82, fill="#091827", outline="")
        c.create_line(0, 81, width, 81, fill="#203649")

        c.create_text(28, 24, anchor="w", text="STORM WATCH",
                      fill="#f3f7fa", font=("Segoe UI", 21, "bold"))
        c.create_text(28, 52, anchor="w",
                      text="MEXICO • HURRICANE MONITORING SYSTEM",
                      fill="#718b9c", font=("Segoe UI", 8, "bold"))

        live_color = "#f3c969" if self.loading else "#45d19b"
        c.create_oval(width - 160, 27, width - 152, 35, fill=live_color, outline="")
        c.create_text(width - 143, 31, anchor="w",
                      text="LIVE DATA",
                      fill="#b7c6cf", font=("Segoe UI", 8, "bold"))

        margin = 24
        top = 104
        bottom = height - 25
        left_width = 330
        gap = 18
        lx1 = margin
        lx2 = lx1 + left_width
        rx1 = lx2 + gap
        rx2 = width - margin

        self.rounded_rect(c, lx1, top, lx2, bottom,
                          radius=16, fill="#0d1a28", outline="#1e3548")
        self.rounded_rect(c, rx1, top, rx2, bottom,
                          radius=16, fill="#0a1622", outline="#1e3548")

        c.create_text(lx1 + 22, top + 23, anchor="w",
                      text="CURRENT STATUS",
                      fill="#728b9b", font=("Segoe UI", 8, "bold"))

        status = self.data["status"]
        if self.loading:
            title = "CONNECTING"
            subtitle = "Loading NHC data..."
        elif status == "hurricane":
            title = "HURRICANE DETECTED"
            subtitle = self.data["name"]
        elif status == "calm":
            title = "NO HURRICANES"
            subtitle = "Monitoring area is calm"
        else:
            title = "DATA ERROR"
            subtitle = "Check your internet connection"

        dot = self.status_color()
        c.create_oval(lx1 + 22, top + 52, lx1 + 34, top + 64, fill=dot, outline="")
        c.create_text(lx1 + 45, top + 58, anchor="w", text=title,
                      fill="#edf4f7", font=("Segoe UI", 13, "bold"))
        c.create_text(lx1 + 22, top + 87, anchor="w", text=subtitle,
                      fill="#9eb2bf", font=("Segoe UI", 10))

        c.create_line(lx1 + 22, top + 112, lx2 - 22, top + 112, fill="#1f3547")

        self.draw_wind_gauge((lx1 + lx2) / 2, top + 230, 92)

        detail_y = top + 370
        c.create_text(lx1 + 22, detail_y, anchor="w",
                      text="STORM DETAILS",
                      fill="#728b9b", font=("Segoe UI", 8, "bold"))

        rows = [
            ("COORDINATES",
             (f"{self.data['lat']:.1f}°, {self.data['lon']:.1f}°"
              if self.data["lat"] is not None else "Mexico region")),
            ("WIND DIRECTION",
             (f"{self.data['direction']:.0f}°"
              if self.data["direction"] is not None else "no data")),
            ("UPDATED",
             time.strftime("%H:%M:%S", time.localtime(self.last_refresh))
             if self.last_refresh else "--:--:--")
        ]

        y = detail_y + 31
        for label, value in rows:
            c.create_text(lx1 + 22, y, anchor="w", text=label,
                          fill="#617989", font=("Segoe UI", 7, "bold"))
            c.create_text(lx2 - 22, y, anchor="e", text=value,
                          fill="#d4e0e6", font=("Segoe UI", 9, "bold"))
            y += 35

        bx1 = lx1 + 22
        bx2 = lx2 - 22
        by2 = bottom - 23
        by1 = by2 - 43
        self.rounded_rect(c, bx1, by1, bx2, by2,
                          radius=9, fill="#15354a", outline="#2e5d78")
        c.create_text((bx1 + bx2) / 2, (by1 + by2) / 2,
                       text="↻   REFRESH DATA",
                       fill="#e8f1f5", font=("Segoe UI", 9, "bold"))
        self.refresh_box = (bx1, by1, bx2, by2)

        c.create_text(rx1 + 22, top + 22, anchor="w",
                      text="REAL-TIME SIMULATION",
                      fill="#728b9b", font=("Segoe UI", 8, "bold"))
        c.create_text(rx1 + 22, top + 48, anchor="w",
                      text="VEGETATION RESPONSE",
                      fill="#e8f0f4", font=("Segoe UI", 13, "bold"))

        scene_top = top + 70
        scene_bottom = bottom - 72
        self.scene_box = (rx1 + 16, scene_top, rx2 - 16, scene_bottom)

        self.draw_scene_background(
            rx1 + 16, scene_top, rx2 - 16, scene_bottom
        )

        c.create_text(rx1 + 22, bottom - 51, anchor="w",
                      text=self.data["note"],
                      fill="#8fa5b2", font=("Segoe UI", 8),
                      width=(rx2 - rx1) - 44)
        c.create_text(rx2 - 22, bottom - 18, anchor="e",
                      text="NHC / OPEN-METEO",
                      fill="#526c7c", font=("Segoe UI", 7, "bold"))

        self.rain_initialized = False
        self.rain.clear()
        self.draw_tree_frame()

    def draw_scene_background(self, x1, y1, x2, y2):
        c = self.canvas
        w = x2 - x1
        h = y2 - y1
        active = self.data["status"] == "hurricane"
        strength = self.wind_strength()

        if active:
            sky_top = "#111c2b"
            sky_bottom = "#26384a"
            cloud = "#3d4a58"
        else:
            sky_top = "#24445a"
            sky_bottom = "#5d8292"
            cloud = "#91a9b5"

        steps = 18
        for i in range(steps):
            ratio = i / (steps - 1)
            r1 = int(sky_top[1:3], 16)
            g1 = int(sky_top[3:5], 16)
            b1 = int(sky_top[5:7], 16)
            r2 = int(sky_bottom[1:3], 16)
            g2 = int(sky_bottom[3:5], 16)
            b2 = int(sky_bottom[5:7], 16)
            rr = int(r1 + (r2 - r1) * ratio)
            gg = int(g1 + (g2 - g1) * ratio)
            bb = int(b1 + (b2 - b1) * ratio)
            color = f"#{rr:02x}{gg:02x}{bb:02x}"
            yy1 = y1 + h * i / steps
            yy2 = y1 + h * (i + 1) / steps
            c.create_rectangle(x1, yy1, x2, yy2, fill=color, outline="")

        cloud_speed = self.phase * (5 if active else 1.0)
        cloud_groups = [
            (0.18, 0.20, 55),
            (0.43, 0.15, 70),
            (0.72, 0.23, 60),
            (0.90, 0.16, 48)
        ]
        for index, (cx, cy, radius) in enumerate(cloud_groups):
            radius = min(radius, w * 0.13)
            drift = (math.sin(self.phase * 0.08 + index) * (8 + strength * 25)
                     - cloud_speed * 0.001 * index)
            cloud_x = x1 + w * cx + drift
            cloud_y = y1 + h * cy
            pieces = [
                (-0.70, 0.08, 0.75), (-0.35, -0.30, 0.90),
                (0.00, -0.42, 1.00), (0.36, -0.27, 0.88),
                (0.70, 0.08, 0.70)
            ]
            for px, py, scale in pieces:
                rr = radius * scale
                c.create_oval(
                    cloud_x + px * radius - rr,
                    cloud_y + py * radius - rr * 0.50,
                    cloud_x + px * radius + rr,
                    cloud_y + py * radius + rr * 0.50,
                    fill=cloud, outline=""
                )

        ground_y = y1 + h * 0.80
        c.create_rectangle(x1, ground_y - 18, x2, ground_y,
                           fill="#496a66" if active else "#70948a", outline="")

        c.create_rectangle(x1, ground_y, x2, y2,
                           fill="#213f35" if active else "#315844", outline="")

        direction_x, _ = self.wind_direction_vector()
        grass_color = "#3e6751" if active else "#4e7855"
        for i in range(60):
            gx = x1 + 5 + i * max(2, int(w / 60))
            gust = self.data.get("gust", 0) or 0
            bend = direction_x * (3 + min(16, gust * 0.07))
            gy = ground_y + random.uniform(1, 10)
            c.create_line(gx, gy + 8, gx + bend, gy - 1,
                          fill=grass_color, width=1)

        for i in range(10):
            bx = x1 + i * w / 9
            by = ground_y + 2
            rr = 8 + (i % 3) * 4
            c.create_oval(bx - rr, by - rr, bx + rr, by + rr,
                         fill="#1e4938", outline="")

        c.create_text(x1 + 18, y2 - 15, anchor="w",
                      text="LIVE WIND RESPONSE",
                      fill="#c7d8dd", font=("Segoe UI", 8, "bold"))

    def draw_tree_frame(self):
        if not self.scene_box:
            return
        self.canvas.delete("tree")
        self.draw_detailed_tree_from_scene()

    def draw_detailed_tree_from_scene(self):
        x1, y1, x2, y2 = self.scene_box
        w = x2 - x1
        h = y2 - y1
        ground_y = y1 + h * 0.80
        wind = self.data["wind"] if self.data["status"] == "hurricane" else 0.0
        gust = self.data["gust"] if self.data["status"] == "hurricane" else 0.0
        direction = self.data["direction"]
        tree_height = min(h * 0.64, w * 0.55)
        self.draw_realistic_tree(x1 + w * 0.50, ground_y + 4,
                                 tree_height, wind, gust, direction)

    def draw_realistic_tree(self, cx, ground_y, tree_h, wind, gust, direction):
        c = self.canvas
        strength = min(1.0, max(wind, gust) / 180.0)
        direction_x, _ = self.wind_direction_vector()

        if strength <= 0.01:
            target_angle = 0.0
        else:
            normalized = min(1.0, strength)
            target_angle = (math.radians(7) + math.radians(34) * (normalized ** 1.35))
            target_angle *= direction_x

        gust_factor = min(1.0, max(0.0, gust / 150.0))
        slow_wave = math.sin(self.phase * (1.15 + strength * 1.5))
        fast_wave = math.sin(self.phase * 3.3 + 1.7) * 0.35
        irregular = (slow_wave * 0.70 + fast_wave +
                    math.sin(self.phase * 0.47) * 0.20)
        sway = (math.radians(2.0) + math.radians(7.0) * gust_factor) * irregular
        angle = target_angle + sway
        bend_power = 1.45 + strength * 0.55

        def centerline(fraction):
            y = ground_y - tree_h * fraction
            curve = (fraction ** bend_power * math.sin(angle) * tree_h * 0.48)
            secondary = (math.sin(fraction * math.pi) * math.sin(angle * 1.7)
                         * tree_h * 0.035 * strength)
            x = cx + curve + secondary
            return x, y

        shadow_scale = 1.0 + strength * 0.35
        c.create_oval(cx - 105 * shadow_scale, ground_y - 7,
                      cx + 105 * shadow_scale, ground_y + 18,
                      fill="#142823", outline="", tags="tree")

        root_x, root_y = centerline(0.02)
        for side in (-1, 1):
            root_length = 34 + strength * 12
            c.create_line(root_x, root_y, root_x + side * root_length,
                          ground_y + 5, fill="#493626", width=8,
                          capstyle=tk.ROUND, tags="tree")

        levels = [0.00, 0.10, 0.20, 0.32, 0.44, 0.56, 0.68, 0.80, 0.90, 1.00]
        base_width = max(16, min(30, tree_h * 0.055))
        widths = [max(3, base_width * (0.30 + 0.72 * (1.0 - f))) for f in levels]

        for i in range(len(levels) - 1):
            ax, ay = centerline(levels[i])
            bx, by = centerline(levels[i + 1])
            width = widths[i]
            c.create_line(ax, ay, bx, by, fill="#39291f", width=width + 6,
                          capstyle=tk.ROUND, tags="tree")
            c.create_line(ax - 1, ay, bx - 1, by, fill="#765339", width=width,
                          capstyle=tk.ROUND, tags="tree")
            c.create_line(ax - width * 0.22, ay, bx - width * 0.22, by,
                          fill="#a87950", width=max(2, width * 0.22),
                          capstyle=tk.ROUND, tags="tree")
            c.create_line(ax + width * 0.22, ay, bx + width * 0.22, by,
                          fill="#4e3829", width=max(1, width * 0.11),
                          capstyle=tk.ROUND, tags="tree")

        branch_specs = [
            (0.40, -1, 0.24, 0.38), (0.49, 1, 0.28, 0.35),
            (0.59, -1, 0.27, 0.32), (0.69, 1, 0.25, 0.29),
            (0.78, -1, 0.21, 0.25), (0.86, 1, 0.18, 0.22),
            (0.92, -1, 0.13, 0.18)
        ]
        branch_tips = []
        for fraction, side, length, rise in branch_specs:
            bx, by = centerline(fraction)
            branch_length = tree_h * length
            raw_x = side * branch_length
            raw_y = -tree_h * rise
            local_angle = angle * (0.55 + fraction * 0.45)
            tip_x = bx + raw_x * math.cos(local_angle) - raw_y * math.sin(local_angle)
            tip_y = by + raw_x * math.sin(local_angle) + raw_y * math.cos(local_angle)
            branch_tips.append((tip_x, tip_y, fraction))
            branch_width = max(3, tree_h * 0.020 * (1.2 - fraction * 0.45))
            c.create_line(bx, by, tip_x, tip_y, fill="#39291f",
                          width=branch_width + 3, capstyle=tk.ROUND, tags="tree")
            c.create_line(bx, by, tip_x, tip_y, fill="#684a34",
                          width=branch_width, capstyle=tk.ROUND, tags="tree")
            extension = 0.30
            ex = tip_x + (tip_x - bx) * extension + direction_x * tree_h * 0.025 * strength
            ey = tip_y + (tip_y - by) * extension
            c.create_line(tip_x, tip_y, ex, ey, fill="#60432f",
                          width=max(2, branch_width * 0.55),
                          capstyle=tk.ROUND, tags="tree")
            branch_tips.append((ex, ey, fraction))

        crown_centers = [centerline(0.98), centerline(0.90), centerline(0.82),
                        centerline(0.73), centerline(0.64)]
        for tip_x, tip_y, _ in branch_tips:
            crown_centers.append((tip_x, tip_y))

        leaf_colors = ["#173c2d", "#1d4b34", "#245b3b", "#2d6942",
                       "#39764a", "#4b8754", "#5b945d"]
        compression = 1.0 - strength * 0.24

        for group_index, center in enumerate(crown_centers):
            center_x, center_y = center
            base_radius = tree_h * (0.105 if group_index < 5 else 0.082)
            local_fraction = min(1.0, 0.55 + group_index * 0.045)
            extra_dx = direction_x * strength * tree_h * 0.045 * local_fraction
            center_x += extra_dx
            leaf_shapes = [
                (-0.56, 0.03, 0.78, 0.66), (-0.30, -0.25, 0.88, 0.73),
                (0.00, -0.34, 0.95, 0.78), (0.32, -0.23, 0.88, 0.72),
                (0.57, 0.03, 0.78, 0.66), (-0.18, 0.22, 0.88, 0.70),
                (0.20, 0.21, 0.86, 0.69)
            ]
            for leaf_index, (dx, dy, sx, sy) in enumerate(leaf_shapes):
                local_x = dx * base_radius * compression
                local_y = dy * base_radius
                micro = math.sin(self.phase * 2.3 + group_index * 0.7 + leaf_index * 0.4)
                micro_shift = micro * strength * tree_h * 0.008
                px = (center_x + local_x * math.cos(angle) - local_y * math.sin(angle)
                      + direction_x * micro_shift)
                py = (center_y + local_x * math.sin(angle) + local_y * math.cos(angle))
                leaf_angle = (angle * (0.65 + group_index * 0.025) +
                              sway * (0.35 + leaf_index * 0.08))
                rx = base_radius * sx * compression
                ry = base_radius * sy
                coords = []
                for pi in range(22):
                    t = 2 * math.pi * pi / 22
                    edge = 1.0 + 0.06 * math.sin(t * 3 + group_index + leaf_index)
                    lx = rx * edge * math.cos(t)
                    ly = ry * edge * math.sin(t)
                    coords.extend([
                        px + lx * math.cos(leaf_angle) - ly * math.sin(leaf_angle),
                        py + lx * math.sin(leaf_angle) + ly * math.cos(leaf_angle)
                    ])
                color_index = (group_index * 2 + leaf_index) % len(leaf_colors)
                c.create_polygon(coords, fill=leaf_colors[color_index],
                                 outline="", smooth=True, tags="tree")
                if leaf_index in (1, 3, 5):
                    hx = px - rx * 0.20
                    hy = py - ry * 0.22
                    c.create_oval(hx - 3, hy - 2, hx + 3, hy + 2,
                                  fill="#76a66c", outline="", tags="tree")

    def on_click(self, event):
        x1, y1, x2, y2 = self.refresh_box
        if x1 <= event.x <= x2 and y1 <= event.y <= y2:
            self.refresh()

    def on_resize(self, _event):
        if self.root.winfo_exists():
            self.draw_static()

    def animate(self):
        if not self.root.winfo_exists():
            return
        now = time.perf_counter()
        delta = now - self.last_frame_time
        delta = min(delta, 0.05)
        self.last_frame_time = now
        self.phase += delta * 3.5
        self.wind_phase += delta

        self.draw_tree_frame()

        self.update_rain(delta)
        self.canvas.delete("rain")
        self.draw_rain(self.canvas)

        self.trigger_lightning()
        self.canvas.delete("lightning")
        if self.lightning_active:
            elapsed = time.time() - self.lightning_timer
            if elapsed < self.lightning_duration:
                if self.scene_box:
                    x1, y1, x2, y2 = self.scene_box
                    self.canvas.create_rectangle(x1, y1, x2, y2,
                                                  fill="#b8d1d8", stipple="gray25",
                                                  outline="", tags="lightning")
                    lx = x1 + random.uniform(0.20, 0.80) * (x2 - x1)
                    self.draw_lightning(self.canvas, lx, y1,
                                        lx + random.uniform(-60, 60),
                                        y1 + (y2 - y1) * 0.50)
            else:
                self.lightning_active = False

        self.debris_spawn_timer += delta
        if (self.data["status"] == "hurricane" and
                self.debris_spawn_timer > random.uniform(0.12, 0.38)):
            self.debris_spawn_timer = 0
            self.spawn_debris()
        self.update_debris(delta)
        self.canvas.delete("debris")
        self.draw_debris(self.canvas)

        if (not self.loading and
                time.time() - self.last_refresh >= REFRESH_SECONDS):
            self.refresh()

        self.root.after(FRAME_DELAY_MS, self.animate)


if __name__ == "__main__":
    root = tk.Tk()
    app = MexicoHurricaneTree(root)
    root.mainloop()

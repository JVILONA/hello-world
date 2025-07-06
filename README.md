import requests
import pytesseract
from PIL import Image, ImageDraw, ImageFont
import time

# Optional: set this to True if you want it to recheck every 5 seconds
REPEAT = False

# URL to the published Google Doc (plain text export)
DOC_URL = "https://docs.google.com/document/d/e/2PACX-1vTER-wL5E8YC9pxDx43gk8eIds59GtUUk4nJo_ZWagbnrH0NFvMXIw6VWFLpf5tWTZIT9P9oLIoFJ6A/pub?format=txt"

def fetch_doc_lines(url):
    try:
        response = requests.get(url)
        response.raise_for_status()
        return response.text.strip().splitlines()
    except Exception as e:
        print("Error fetching document:", e)
        return []

def parse_grid(lines):
    grid_data = {}
    max_x = 0
    max_y = 0
    i = 0
    while i + 2 < len(lines):
        try:
            x = int(lines[i].strip())
            char = lines[i+1].strip()
            y = int(lines[i+2].strip())
            grid_data[(x, y)] = char
            max_x = max(max_x, x)
            max_y = max(max_y, y)
        except:
            pass
        i += 3
    return grid_data, max_x, max_y

def build_grid_string(grid_data, max_x, max_y):
    grid_lines = []
    for y in range(max_y + 1):
        line = ""
        for x in range(max_x + 1):
            line += grid_data.get((x, y), " ")
        grid_lines.append(line)
    return "\n".join(grid_lines)

def render_text_to_image(text, font_size=28):
    font = ImageFont.load_default()
    width = max(len(line) for line in text.splitlines()) * font_size // 2
    height = len(text.splitlines()) * font_size

    img = Image.new("L", (width, height), color=255)
    draw = ImageDraw.Draw(img)
    y_offset = 0
    for line in text.splitlines():
        draw.text((0, y_offset), line, font=font, fill=0)
        y_offset += font_size
    return img

def extract_message_from_grid(grid_text):
    img = render_text_to_image(grid_text)
    ocr_result = pytesseract.image_to_string(img, config="--psm 8 -c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ")
    return ocr_result.strip()

def run_once():
    lines = fetch_doc_lines(DOC_URL)
    if not lines:
        print("No data.")
        return

    grid_data, max_x, max_y = parse_grid(lines)
    grid_text = build_grid_string(grid_data, max_x, max_y)

    print("\n🧱 Unicode Grid:\n")
    print(grid_text)

    secret_message = extract_message_from_grid(grid_text)
    print(f"\n🔐 Secret Message: {secret_message or '[Not detected]'}")

# --- Main Loop ---
if REPEAT:
    while True:
        run_once()
        print("\n🔄 Refreshing in 5 seconds...\n")
        time.sleep(5)
else:
    run_once()

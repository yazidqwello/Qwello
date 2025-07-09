# Qwello
import random
import datetime
import json
import os
from slack_sdk import WebClient
from slack_sdk.errors import SlackApiError

participants = ["Yazid", "Ayoub", "Vincent", "Sorel", "Jessica", "jamila"]
channel_name = "#teststandup"
slack_token = os.getenv("SLACK_BOT_TOKEN")

today = datetime.datetime.today()
weekday = today.weekday()  # 0 = lundi, 4 = vendredi

if weekday > 4:
    print("C'est le week-end, pas de standup.")
    exit()

# Fichier contenant l'ordre de passage tournant
order_file = "schedule_order.json"
used_file = "used_names.json"

# Règle : disponibilité spécifique
def is_available(name, weekday):
    if name == "jamila":
        return weekday in [0, 1]
    return True

# Charger ou initialiser l'ordre
if os.path.exists(order_file):
    with open(order_file, "r") as f:
        rotation = json.load(f)
else:
    rotation = participants.copy()
    random.shuffle(rotation)  # première semaine : aléatoire
    with open(order_file, "w") as f:
        json.dump(rotation, f)

# Réinitialiser la semaine chaque lundi
if weekday == 0:
    if os.path.exists(used_file):
        os.remove(used_file)
    # faire tourner l’ordre
    rotation = rotation[1:] + [rotation[0]]
    with open(order_file, "w") as f:
        json.dump(rotation, f)

# Charger les noms déjà passés cette semaine
if os.path.exists(used_file):
    with open(used_file, "r") as f:
        used_names = json.load(f)
else:
    used_names = []

# Choisir la première personne dans la rotation encore dispo aujourd’hui
for name in rotation:
    if name not in used_names and is_available(name, weekday):
        selected = name
        used_names.append(selected)
        with open(used_file, "w") as f:
            json.dump(used_names, f)
        message = f"🎤 Bonjour ! Aujourd'hui, c'est au tour de *{selected}* de faire le standup."
        break
else:
    message = "🚫 Aucun participant disponible aujourd’hui."

# Envoi Slack
client = WebClient(token=slack_token)

try:
    client.chat_postMessage(channel=channel_name, text=message)
    print("Message Slack envoyé.")
except SlackApiError as e:
    print(f"Erreur Slack : {e.response['error']}")

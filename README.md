# Smart-Study-Mate
Smart Study Mate automates the study planning process for students. It covers planning, scheduling, summarizing notes, and setting reminders. This Kaggle-ready notebook showcases a local agent simulation that models ADK concepts: Tools, Memory, Planning, and API Calls.
# Core implementation: Helpers, LocalAgent simulation (Tools, Memory, Planning, API calls)

import os, json, time
from datetime import datetime, timedelta
from typing import List, Optional, Dict, Any

import requests

MEMORY_PATH = '/kaggle/working/agent_memory.json'

def load_memory(path: str = MEMORY_PATH) -> Dict[str, Any]:
    if os.path.exists(path):
        with open(path, 'r') as f:
            return json.load(f)
    return {"tasks": [], "preferences": {}, "history": []}

def save_memory(mem: Dict[str, Any], path: str = MEMORY_PATH):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, 'w') as f:
        json.dump(mem, f, indent=2, default=str)

def create_task(title: str, due: Optional[str] = None, tags: Optional[List[str]] = None) -> Dict[str, Any]:
    return {
        "id": int(time.time() * 1000),
        "title": title,
        "due": due,
        "tags": tags or [],
        "created_at": datetime.utcnow().isoformat()
    }

def plan_study_session(topic: str, duration_minutes: int = 60) -> List[str]:
    steps = [
        f"Set learning objective for '{topic}' (5 mins)",
        f"Quick pre-test / recall on {topic} (10 mins)",
        f"Focused study of core concepts in {topic} (30 mins)",
        f"Practice problems / exercises (10 mins)",
        f"Quick summary / flashcard creation (5 mins)",
    ]
    if duration_minutes < 60:
        return [s for s in steps if 'Focused' in s or 'summary' in s or 'objective' in s]
    return steps

def get_weather_for_city(city: str) -> Dict[str, Any]:
    try:
        geocode = requests.get(
            "https://nominatim.openstreetmap.org/search",
            params={"q": city, "format": "json", "limit": 1},
            headers={"User-Agent": "KaggleAgent/1.0"},
            timeout=10
        ).json()
        if not geocode:
            return {"error": "Location not found"}
        lat = geocode[0]['lat']; lon = geocode[0]['lon']
        weather = requests.get(
            "https://api.open-meteo.com/v1/forecast",
            params={"latitude": lat, "longitude": lon, "current_weather": True},
            timeout=10
        ).json()
        return weather
    except Exception as e:
        return {"error": str(e)}

# LocalAgent simulates ADK behavior
class LocalAgent:
    def __init__(self, memory_path: str = MEMORY_PATH):
        self.memory = load_memory(memory_path)
        self.memory_path = memory_path

    def tool_create_task(self, title: str, due: Optional[str] = None, tags: Optional[List[str]] = None):
        task = create_task(title, due, tags)
        self.memory.setdefault('tasks', []).append(task)
        save_memory(self.memory, self.memory_path)
        return task

    def tool_list_tasks(self):
        return self.memory.get('tasks', [])

    def tool_get_weather(self, city: str):
        return get_weather_for_city(city)

    def plan_and_execute(self, user_request: str) -> Dict[str, Any]:
        plan = []; actions = []
        if 'study' in user_request.lower() or 'learn' in user_request.lower():
            parts = user_request.split('for')
            topic = parts[0].replace('study','').replace('learn','').strip() or 'Unknown Topic'
            duration = 60
            plan = plan_study_session(topic, duration)
            task = self.tool_create_task(f"Study: {topic}", due=(datetime.utcnow() + timedelta(days=1)).isoformat())
            actions.append({'action':'create_task','result':task})
        elif 'weather' in user_request.lower():
            city = user_request.split('in')[-1].strip() if 'in' in user_request else 'New York'
            weather = self.tool_get_weather(city)
            actions.append({'action':'get_weather','result':weather})
        else:
            plan = ["1) Clarify request","2) Plan steps","3) Execute tools","4) Save to memory"]

        self.memory.setdefault('history',[]).append({
            'timestamp': datetime.utcnow().isoformat(),
            'request': user_request,
            'plan': plan,
            'actions': actions
        })
        save_memory(self.memory, self.memory_path)
        return {'plan': plan, 'actions': actions}

# Create agent instance for demo
agent = LocalAgent()
print("LocalAgent initialized. Memory file at:", MEMORY_PATH)

# Demo: run three example interactions (these will attempt HTTP calls; may be blocked on Kaggle execution)
print("\n--- Demo 1: Study Plan (Linear Algebra) ---")
res1 = agent.plan_and_execute("Study Linear Algebra for exam prep")
print("Plan:")
for s in res1['plan']:
    print("-", s)
print("\nActions:")
print(res1['actions'])

print("\n--- Demo 2: Weather (Mumbai) ---")
res2 = agent.plan_and_execute("What is the weather in Mumbai")
print("Actions:")
print(res2['actions'] if res2['actions'] else res2)

print("\n--- Demo 3: Create Task ---")
res3 = agent.tool_create_task("Practice coding problems", due=(datetime.utcnow()+timedelta(days=2)).isoformat())
print("Created Task:")
print(res3)

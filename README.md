 # UNIproject1
 # Making a project for university assignment 
# Author - Yash Kumar

import requests
import json
import time

# --- Configuration and Constants ---
# NOTE: In a real-world scenario, you would securely load the API key from an environment variable.
# For this demonstration, we use an empty string as required by the Canvas environment.


# The expected structured JSON output enforced by the LLM
WEATHER_SCHEMA = {
    "type": "OBJECT",
    "properties": {
        "city": {"type": "STRING"},
        "temperature_celsius": {"type": "STRING"},
        "condition": {"type": "STRING"},
        "humidity_percent": {"type": "STRING"},
        "wind_speed_kph": {"type": "STRING"}
    },
    "required": ["city", "temperature_celsius", "condition", "humidity_percent", "wind_speed_kph"]
}
 
def get_weather_icon(condition: str) -> str:

    """Maps weather condition keywords to icons (emojis)."""
    lower_condition = condition.lower()
    if 'rain' in lower_condition: return '🌧'
    if 'snow' in lower_condition: return '❄'
    if 'cloud' in lower_condition: return '☁'
    if 'sun' in lower_condition or 'clear' in lower_condition: return '☀'
    if 'fog' in lower_condition or 'mist' in lower_condition: return '🌫'
    if 'thunder' in lower_condition: return '⛈'
    return '🌡' # Default

def fetch_weather_data(city: str) -> tuple:
    """
    Fetches real-time weather data for a given city using the Gemini API 
    with Google Search grounding and structured output.
    
    Args:
        city: The name of the city to query.
        
    Returns:
        A tuple containing the weather data (dict) and source attributions (list), 
        or (None, None) if the request fails.
    """
    print(f"\n🔍 Searching for current weather in {city}...")
    
    user_query = f"What is the current weather, temperature, humidity, and wind speed in {city}? The data must be current and verifiable by Google Search."
    
    payload = {
        "contents": [{"parts": [{"text": user_query}]}],
        # NFR: Reliability - Use Google Search for real-time grounding
        "tools": [{"google_search": {}}],
        "systemInstruction": {
            "parts": [{"text": "You are a specialized weather API. Your only task is to find the current weather conditions for the requested city using search and return the result strictly as a JSON object matching the provided schema. If data is not found, return the JSON with 'Not Found' values for all fields."}]
        },
        "generationConfig": {
            "responseMimeType": "application/json",
            "responseSchema": WEATHER_SCHEMA
        }
    }

    headers = {'Content-Type': 'application/json'}
    
    # Implement exponential backoff (NFR: Error Handling)
    max_retries = 3
    for attempt in range(max_retries):
        try:
            response = requests.post(API_URL, headers=headers, data=json.dumps(payload))
            response.raise_for_status() # Raises an HTTPError if the status is 4xx or 5xx

            result = response.json()
            candidate = result.get('candidates', [{}])[0]

            if not candidate or not candidate.get('content', {}).get('parts', [{}])[0].get('text'):
                print(f"⚠ Attempt {attempt + 1}: LLM returned an empty or invalid response structure.")
                continue

            # 1. Extract and parse the structured JSON
            json_text = candidate['content']['parts'][0]['text']
            weather_data = json.loads(json_text)
            
            # 2. Extract grounding sources
            sources = []
            grounding_metadata = candidate.get('groundingMetadata')
            if grounding_metadata and grounding_metadata.get('groundingAttributions'):
                sources = [
                    {"uri": attr.get('web', {}).get('uri'), "title": attr.get('web', {}).get('title')}
                    for attr in grounding_metadata['groundingAttributions']
                    if attr.get('web', {}).get('uri') and attr.get('web', {}).get('title')
                ]
                
            return weather_data, sources

        except requests.exceptions.HTTPError as e:
            if 500 <= e.response.status_code < 600 and attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Server error ({e.response.status_code}). Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                print(f"❌ HTTP Error: Could not connect to the API. Status Code: {e.response.status_code}")
                return None, None
        except requests.exceptions.RequestException as e:
            print(f"❌ Connection Error: {e}")
            return None, None
        except json.JSONDecodeError:
            print(f"❌ Error: Failed to parse JSON response from LLM (Attempt {attempt + 1}).")
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                return None, None
        
    return None, None


def display_weather(data: dict, sources: list):
    """
    FR3: Displays the fetched weather data and grounding sources in a clean format.
    """
    print("-" * 50)
    
    city = data.get('city', 'Unknown City')
    
    # Check for LLM 'Not Found' response
    if 'not found' in city.lower():
        print(f"💔 WeatherWise: Data not found for the requested city.")
        print("-" * 50)
        return

    # Dynamic Display
    condition = data.get('condition', 'N/A')
    icon = get_weather_icon(condition)
    
    print(f"🌍 Current Weather in {city}")
    print(f"   {icon} {condition}")
    print(f"   ------------------------------")
    print(f"   🌡 Temperature: {data.get('temperature_celsius', 'N/A')}")
    print(f"   💧 Humidity:    {data.get('humidity_percent', 'N/A')}")
    print(f"   💨 Wind Speed:  {data.get('wind_speed_kph', 'N/A')}")
    
    # NFR: Reliability - Display Sources
    print("\n   ✅ Data Reliability (Grounding Sources):")
    if sources:
        for i, source in enumerate(sources):
            # Limiting the source title length for clean CLI output
            title = source['title'][:60] + '...' if len(source['title']) > 60 else source['title']
            print(f"   - Source {i+1}: {title} ({source['uri']})")
    else:
        print("   - No specific grounding sources were found for verification.")
        
    print("-" * 50)


def main():
    """Main application loop (FR1: User Input)."""
    print("=" * 50)
    print("    WeatherWise: Real-Time Weather Detector CLI")
    print("=" * 50)

    while True:
        try:
            city_input = input("Enter city name (or 'quit' to exit): ").strip()
            
            if city_input.lower() == 'quit':
                print("👋 Thank you for using WeatherWise. Goodbye!")
                break
                
            if not city_input:
                print("⚠ Please enter a valid city name.")
                continue

            weather_data, sources = fetch_weather_data(city_input)
            
            if weather_data:
                display_weather(weather_data, sources)

        except KeyboardInterrupt:
            print("\n👋 Exiting WeatherWise. Goodbye!")
            break

# Ensure the main function is called when the script runs
if _name_ == "_main_":
    main()
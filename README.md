//Python-Voice-Assistant
//A Python-based AI Voice Assistant with Wikipedia and WolframAlpha integration
import sounddevice as sd
import numpy as np
import speech_recognition as sr
import pyttsx3
import webbrowser
import wikipedia
import wolframalpha
from datetime import datetime
import os

# Initialize text-to-speech engine
engine = pyttsx3.init()
voices = engine.getProperty('voices')
engine.setProperty('voice', voices[1].id)  # 0 = male, 1 = female
activationWord = 'computer'  # Single word

# Register Chrome as the default browser
chrome_path = r"C:\Program Files\Google\Chrome\Application\chrome.exe"
webbrowser.register('chrome', None, webbrowser.BackgroundBrowser(chrome_path))

# Wolfram Alpha client

appId = os.getenv("WOLFRAM_APP_ID")
wolframClient = wolframalpha.Client(appId)

def speak(text, rate=120):
    engine.setProperty('rate', rate)
    engine.say(text)
    engine.runAndWait()

def parseCommand():
    listener = sr.Recognizer()
    print('Listening for a command...')
    
    # Define sample rate and duration
    sample_rate = 16000  # Common sample rate for speech recognition
    duration = 5  # Duration of the recording in seconds
    
    # Record audio using sounddevice
    print("Recording...")
    audio_data = sd.rec(int(sample_rate * duration), samplerate=sample_rate, channels=1, dtype='int16')
    sd.wait()  # Wait for the recording to finish
    
    # Convert the NumPy array to bytes
    audio_bytes = audio_data.tobytes()
    audio = sr.AudioData(audio_bytes, sample_rate, 2)  # 2 is for mono channel
    
    try:
        print('Recognizing speech...')
        query = listener.recognize_google(audio, language='en_gb')
        print(f'The input speech was: {query}')
        return query
    except sr.UnknownValueError:
        print("I could not understand the audio.")
        speak("I could not understand the audio.")
        return "None"
    except sr.RequestError as e:
        print(f"Could not request results from Google Speech Recognition service; {e}")
        speak("There was an error with the speech recognition service.")
        return "None"

def search_wikipedia(query=''):
    searchResults = wikipedia.search(query)
    if not searchResults:
        print('No Wikipedia results.')
        return 'No result received'
    
    try:
        wikiPage = wikipedia.page(searchResults[0])
    except wikipedia.DisambiguationError as error:
        wikiPage = wikipedia.page(error.options[0])
    print(wikiPage.title)
    return wikiPage.summary

def listOrDict(var):
    if isinstance(var, list):
        return var[0]['plaintext']
    else:
        return var['plaintext']

def search_wolframAlpha(query=''):
    response = wolframClient.query(query)
    if response['@success'] == 'false':
        return 'Could not compute'
    else:
        pod0 = response['pod'][0]
        pod1 = response['pod'][1]
        if (('result' in pod1['@title'].lower()) or 
            (pod1.get('@primary', 'false') == 'true') or 
            ('definition' in pod1['@title'].lower())):
            return listOrDict(pod1['subpod']).split('(')[0]
        else:
            question = listOrDict(pod0['subpod'])
            speak('Computation failed. Querying universal databank.')
            return search_wikipedia(question)

# Main loop
if __name__ == '__main__':
    speak('All systems nominal.')
    while True:
        query = parseCommand().lower().split()
        if query and query[0] == activationWord:
            query.pop(0)
            
            if not query:
                continue
            
            if query[0] == 'say':
                if 'hello' in query:
                    speak('Greetings, all.')
                else:
                    query.pop(0)
                    speech = ' '.join(query)
                    speak(speech)
            
            elif query[0] == 'go' and query[1] == 'to':
                speak('Opening...')
                query = ' '.join(query[2:])
                webbrowser.get('chrome').open_new(query)
            
            elif query[0] == 'wikipedia':
                query = ' '.join(query[1:])
                speak('Querying the universal databank.')
                speak(search_wikipedia(query))
            
            elif query[0] == 'compute':
                query = ' '.join(query[1:])
                speak('Computing...')
                try:
                    result = search_wolframAlpha(query)
                    speak(result)
                except Exception as e:
                    print(f"Error: {e}")
                    speak('Unable to compute.')
            
            elif query[0] == 'log':
                speak('Ready to record your note.')
                newNote = parseCommand().lower()
                now = datetime.now().strftime('%Y-%m-%d-%H-%M-%S')
                with open(f'note_{now}.txt', 'w') as newFile:
                    newFile.write(newNote)
                speak('Note written.')
            
            elif query[0] == 'exit':
                speak('Goodbye.')
                break

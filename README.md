YouTube Notion Updater:


Project Idea:
The YouTube Notion Updater is a tool that tracks YouTube video progress in real-time and updates a Notion database with video details, including:
Video Title
Video URL
Progress (%)
Total Duration (MM:SS)
Last Updated Timestamp (in IST)

This app ensures that all updates for a single video are recorded in the same row in the Notion database, providing seamless real-time tracking.

Technologies Used:

Frontend:
HTML with embedded JavaScript.
YouTube IFrame API for tracking video progress.

Backend:
Flask (Python) for handling requests and communicating with the Notion API.

Database:
Notion database to store video details and progress.

Deployment:
Google Cloud Platform (GCP) for hosting the backend.

Other Tools:
pytz for time zone conversion to IST.
requests library for interacting with the Notion API.

2. Setting Up Notion:
   
Step 1: Create a New Notion API Integration:

Go to the Notion Developers Page.
Click on "Create new integration".
Fill in the details:
Name: YouTube Progress Tracker.
Associated Workspace: Select your workspace where the database will be created.
Copy the generated Integration Token (you’ll need this for your backend).

Step 2: Create a New Notion Database:

Open Notion and create a new database (Table view recommended).

Add the following properties to your database:
Property Name	Property Type	Description
Video Title	Title	The title of the YouTube video being tracked.
Video URL	URL	The URL of the YouTube video being tracked.
Progress	Number	Tracks progress as a percentage (formatted as Percentage in Notion).
Total Duration	Text	The total duration of the video in MM:SS format (stored as rich text).
Last Updated	Date	The timestamp of when the progress was last updated (in IST).

Step 3: Connect Database with API Integration:
Open your Notion database.
Click on the Share button in the top-right corner.
Search for your integration (YouTube Progress Tracker) and invite it to access your database.
Confirm that your integration has access by checking under "Shared With".

4. Step-by-Step Process with Codes and Uses

Step 1: Frontend Code
The frontend is responsible for embedding a YouTube player and sending progress updates to the backend.

File: templates/index.html
xml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>YouTube Progress Tracker</title>
    <script src="https://www.youtube.com/iframe_api"></script>
</head>
<body>
    <h1>YouTube Progress Tracker</h1>
    <div id="player"></div>

    <script>
        let player;
        let lastProgress = 0;

        function onYouTubeIframeAPIReady() {
            player = new YT.Player('player', {
                height: '390',
                width: '640',
                videoId: 'dQw4w9WgXcQ', // Replace with a valid YouTube video ID
                events: {
                    'onStateChange': onPlayerStateChange,
                }
            });
        }

        function onPlayerStateChange(event) {
            if (event.data === YT.PlayerState.PLAYING) {
                setInterval(() => {
                    const progress = Math.floor((player.getCurrentTime() / player.getDuration()) * 100);

                    if (progress >= lastProgress + 1) {
                        lastProgress = progress;
                        fetch('http://127.0.0.1:5001/update', { // Replace with deployed backend URL later
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify({
                                videoTitle: player.getVideoData().title,
                                videoURL: player.getVideoUrl(),
                                progress: progress,
                                totalDuration: Math.floor(player.getDuration())
                            })
                        }).then(response => response.json())
                          .then(data => console.log("Backend Response:", data))
                          .catch(error => console.error("Error:", error));
                    }
                }, 1000);
            }
        }
    </script>
</body>
</html>

Step 2: Backend Code
The backend handles communication between the frontend and Notion API.
File: main.py
python

#Make sure all the following are downloaded

from flask import Flask, render_template, request, jsonify
import requests
from datetime import datetime
from pytz import timezone

app = Flask(__name__)

NOTION_TOKEN = "your_notion_integration_token"
DATABASE_ID = "your_notion_database_id"

headers = {
    "Authorization": f"Bearer {NOTION_TOKEN}",
    "Content-Type": "application/json",
    "Notion-Version": "2022-06-28"
}

@app.route('/')
def home():
    return render_template('index.html')

@app.route('/update', methods=['POST'])
def update_progress():
    data = request.json
    video_title = data.get('videoTitle')
    video_url = data.get('videoURL').strip()
    total_duration_seconds = data.get('totalDuration')
    progress = data.get('progress') / 100

    def convert_seconds_to_duration(total_seconds):
        minutes = total_seconds // 60
        seconds = total_seconds % 60
        return f"{minutes:02d}:{seconds:02d}"

    total_duration_formatted = convert_seconds_to_duration(total_duration_seconds)

    utc_time = datetime.now()
    ist_timezone = timezone('Asia/Kolkata')
    ist_time = utc_time.astimezone(ist_timezone)

    query_payload = {
        "filter": {
            "property": "Video URL",
            "url": {"equals": video_url}
        }
    }
    
    query_response = requests.post(
        f"https://api.notion.com/v1/databases/{DATABASE_ID}/query",
        headers=headers,
        json=query_payload
    )
    
    query_result = query_response.json()

    if query_result["results"]:
        page_id = query_result["results"][0]["id"]
        update_payload = {
            "properties": {
                "Progress": {"number": progress},
                "Last Updated": {"date": {"start": ist_time.isoformat()}}
            }
        }
        update_response = requests.patch(
            f"https://api.notion.com/v1/pages/{page_id}",
            headers=headers,
            json=update_payload
        )
        return jsonify({"status": update_response.status_code, "response": update_response.json()})
    else:
        create_payload = {
            "parent": {"database_id": DATABASE_ID},
            "properties": {
                "Video Title": {"title": [{"text": {"content": video_title}}]},
                "Video URL": {"url": video_url},
                "Total Duration": {"rich_text": [{"text": {"content": total_duration_formatted}}]},
                "Progress": {"number": progress},
                "Last Updated": {"date": {"start": ist_time.isoformat()}}
            }
        }
        create_response = requests.post(
            f"https://api.notion.com/v1/pages",
            headers=headers,
            json=create_payload
        )
        return jsonify({"status_create_row_code:", create_response.status_code, "response": create_response.json()})
 
 if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)
    
4. How to Use and Test

Testing Locally:

Start your Flask app:

bash

python3 main.py

Open your browser and navigate to:

text
http://127.0.0.1:5001/

Play a YouTube video embedded in the page.

Verifying Updates in Notion
Check your Notion database to ensure:
A new row is created when starting playback for a new video.
Progress updates are applied to the same row at 1% intervals.
The final update (100%) overwrites earlier progress values.
Debugging Tips
Use Postman or curl to send test requests to /update with sample data:
json
{
    "videoTitle": "Sample Video",
    "videoURL": "https://youtube.com/sample",
    "progress": 50,
    "totalDuration": 300
}
Check Flask logs for errors or unexpected behavior.
